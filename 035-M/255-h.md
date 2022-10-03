IllIllI
# Accounts may avoid liquidation by using fee-on-transfer tokens that revert when transferring zero amounts

## Summary
Fee-on-transfer tokens deduct a fee after every transfer, meaning the amount transferred is not the amount received. The protocol currently supports tokens that are upgradeable, so there's nothing stopping an upgrade from introducing this behavior.

## Vulnerability Detail
If the fee-on-transfer token also reverts if a balance of zero is attempted to be transferred, an account can transfer just enough of the token, such that the balance after the transfer is zero. When liquidation is attempted, there's no check that the amount being transferred is not zero, and therefore the liquidation will fail when zero is transferred for that token

## Impact
Inability to liquidate will lead to bad debts, leading to insolvency

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/2e25699040ed87a9af62f2b637eafcc6b0b59cc5/protocol/src/core/Account.sol#L163-L174

## Tool used

Manual Review

## Recommendation
Don't transfer an asset if the balance is zero