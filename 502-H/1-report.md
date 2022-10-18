WATCHPUG

# `CurveLPStaking` Curve's gauge rewards cannot be claimed

## Summary

Note: This issue is a part of the extra scope added by Sentiment AFTER the audit contest. This scope was only reviewed by WatchPug and relates to these three PRs:

1. [Lending deposit cap](https://github.com/sentimentxyz/protocol/pull/234)
2. [Fee accrual modification](https://github.com/sentimentxyz/protocol/pull/233)
3. [CRV staking](https://github.com/sentimentxyz/controller/pull/41)

Curve's gauge rewards cannot be claimed.

## Vulnerability Detail

The sole goal of `CurveLPStakingController` is to stake and earn rewards from Curve's gauge system.

However, it doesn't support `claim_rewards()`, and both `deposit()` or `withdraw()` are using the default value (`false`) for `_claim_rewards` parameter.

## Impact

Curve's gauge rewards cannot be claimed.

## Code Snippet
https://github.com/sentimentxyz/controller/blob/be4a7c70ecef788ca9226b46ff108ddd9001b14e/src/curve/CurveLPStakingController.sol#L20-L24

```solidity
/// @notice deposit(uint256)
bytes4 constant DEPOSIT = 0xb6b55f25;

/// @notice withdraw(uint256)
bytes4 constant WITHDRAW = 0x2e1a7d4d;
```

## Tool used

Manual Review

## Recommendation

Consider supporting claim_rewards(address,address) or using deposit(uint256,address,bool) and withdraw(uint256,address,bool) to support claim rewards when deposit() or withdraw().

## Sentiment Team
Fixed controller as recommended to claim gauge rewards and moved gauge token oracle into a separate contract instead. PR [here](https://github.com/sentimentxyz/oracle/pull/42).

## Lead Senior Watson
Confirmed fix. 

