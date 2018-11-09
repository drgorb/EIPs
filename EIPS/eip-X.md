---
eip: X
title: Address and ERC20-compliant transfer rules
author: Cyril Lapinte <cyril.lapinte@mtpelerin.com>, Laurent Aapro <laurent.aapro@mtpelerin.com>
type: Standards Track
category: ERC
status: Draft
created: 2018-11-09
---

## Simple Summary

We propose a standard and an interface to define address and ERC20-compliant transfer rules.
To ease rule reusability and composition, we also propose an interface for a rule engine.

## Abstract

This standard proposal should answer the following challenges:
- Unify address and ERC20-compliant transfer restriction needs required by other EIPs (EIP-902, EIP-1066, EIP-1175, ...)
- Externalizing code and storage, improving altogether reusability, gas costs and contracts memory footprint.
- Expliciting contract behavior and in particular contract behavior change in order to ease user interaction with such contract.
- Enable integration of these rules with interacting platforms such as exchanges or decentralized wallets and DApp.

The last section of this document will illustrate the proposal with an applied use case in ERC20 token as done by Mt Pelerin's Bridge protocol.

## Motivation

ERC20 was designed as a standard interface allowing any tokens on Ethereum to be re-used by other applications: from wallets to decentralized exchanges.
Today, more and more contracts, and in particular tokens, are dealing with regulated assets and services. It is required that they strictly implement regulations. We believe that this standard will serve as the first stone for a comprehensive and universal on-chain regulation framework.

## Specification

## Rule interface

`Rule` interface should provide a way to validate if an address or a transfer is valid.
Very often a transfer validity will be also based on address validity.
It is possible to return true all the time if one of this two methods is not applicable.

```js
pragma solidity ^0.4.24;

interface Rule {
  function isAddressValid(address _address) external view returns (bool);
  function isTransferValid(address _from, address _to, uint256 _amount)
    external view returns (bool);
}
```

## WithRules interface

`WithRules` interface describe the integration of rules to a rule engine.
Developers may choose to not implement this interface if their program will only deal with one rule, or if it is not permitted to update the rules.

The rules ordering must be think through.
Rules less costly to validate or with an higher chance to break should be put first to reduce gas amount.
That is why, rules shall be defined altogether and not individually.

```js
pragma solidity ^0.4.24;

import "./Rule.sol";

interface WithRules {
  function ruleLength() public view returns (uint256);
  function rule(uint256 _ruleId) public view returns (IRule);
  function validateAddress(address _address) public view returns (bool);
  function validateTransfer(address _from, address _to, uint256 _amount)
    public view returns (bool);

  function defineRules(IRule[] _rules) public;

  event RulesDefined(uint256 count);
}
```

## Implementation

Readers may find below an implementation of WithRules engine given as an illustration:

```js
pragma solidity ^0.4.24;

import "../zeppelin/ownership/Ownable.sol";
import "../interface/WithRules.sol";
import "../interface/Rule.sol";

contract WithRules is IWithRules, Ownable {

  IRule[] internal rules;

  /**
   * @dev Constructor
   */
  constructor(IRule[] _rules) public {
    rules = _rules;
  }

  /**
   * @dev Returns the number of rules
   */
  function ruleLength() public view returns (uint256) {
    return rules.length;
  }

  /**
   * @dev Returns the Rule associated to the specified ruleId
   */
  function rule(uint256 _ruleId) public view returns (IRule) {
    return rules[_ruleId];
  }

  /**
   * @dev Check if the rules are valid for an address
   */
  function validateAddress(address _address) public view returns (bool) {
    for (uint256 i = 0; i < rules.length; i++) {
      if (!rules[i].isAddressValid(_address)) {
        return false;
      }
    }
    return true;
  }

  /**
   * @dev Check if the rules are valid
   */
  function validateTransfer(address _from, address _to, uint256 _amount)
    public view returns (bool)
  {
    for (uint256 i = 0; i < rules.length; i++) {
      if (!rules[i].isTransferValid(_from, _to, _amount)) {
        return false;
      }
    }
    return true;
  }
}
```

*** Notes ***
The MPS (Mt Pelerin's Share) token is current live implementation of this standard.
Other implementations may be written with different trade-offs: from gas saving to improved security.

#### Example of rules implementations

- [YesNo rule](https://github.com/MtPelerin/MtPelerin-protocol/tree/master/contracts/rule/YesNoRule.sol): Trivial rule used to demonstrate both a rule and the rule engine.

- [Freeze rule](https://github.com/MtPelerin/MtPelerin-protocol/tree/master/contracts/rule/FreezeRule.sol): This rule allow to prevent all emissions and reception of tokens from chosen addresses.

- [Lock rule](https://github.com/MtPelerin/MtPelerin-protocol/tree/master/contracts/rule/LockRule.sol): Define a global transfer policy preventing either emission or reception of tokens within a period of time. Exception may be provided to some addresses by the rule admin.

- [User Kyc Rule](https://github.com/MtPelerin/MtPelerin-protocol/tree/master/contracts/rule/UserKycRule.sol): Rule example relying on an existing whitelist to assert transfer and addresses validity.

#### Example implementations are available at
- [Mt Pelerin Bridge protocol rules implementation](https://github.com/MtPelerin/MtPelerin-protocol/tree/master/contracts/rule)
- [Mt Pelerin Token with rules](https://github.com/MtPelerin/MtPelerin-protocol/blob/master/contracts/token/component/TokenWithRules.sol)

## History

Historical links related to this standard:

- First swiss regulated tokenized shares issue by Mt Pelerin (MPS token) is using this standard: https://www.mtpelerin.com/blog/world-first-tokenized-shares
After the token issuance and during the tokensale, the rule engine was updated several times to match changing business and legal requirements.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
External references outside this repository will have their own specific copyrights.
