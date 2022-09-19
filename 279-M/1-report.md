GimelSec
# `shares`/`assets` is not yet initialized upon calling `beforeDeposit(assets, shares)` in `ERC4626.deposit()`/`ERC4626.mint()`

## Summary

`shares`/`assets` is not calculated yet in `ERC4626.deposit()`/`ERC4626.mint()` when doing `beforeDeposit(assets, shares)`, leading to a function call with uninitialized parameters

## Vulnerability Detail

In `tokens\utils\ERC4626.sol`, `ERC4626.deposit()`/`ERC4626.mint()` calls `beforeDeposit(assets, shares)` at the beginning. However, `shares`/`assets` is not yet derived from the other at time of function call, resulting in usage of uninitialized variables.

## Impact

Usage of the uninitialized variable in `beforeDeposit` undermines the security of this specific implementation of `ERC4626`. While the current implementation of `LToken` does not rely on the correct value of `shares`/`assets`, this bug foreshadows the security of future contracts that inherit `ERC4626`.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/protocol/src/tokens/utils/ERC4626.sol#L49

## Tool used

Manual Review

## Recommendation

Call `beforeDeposit(assets, shares)` after `shares`/`assets` is calculated.
