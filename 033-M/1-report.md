Lambda
# AccountManager: Liquidations not possible when transfer fails

## Summary
When the transfer of one asset fails, liquidations become impossible.

## Vulnerability Detail
`_liquidate` calls `sweepTo`, which iterates over all assets. When one of those transfers fails, the whole liquidation process therefore fails. There are multiple reasons why a transfer could fail:
1.) Blocked addresses (e.g., USDC)
2.) The balance of the asset is 0, but it is still listed under asset. This can be for instance triggered by performing a 0 value Uniswap swap, in which case it is still added to `tokensIn`. Another way to trigger is to call `deposit` with `amt = 0` (this is another issue that should be fixed IMO, in practice the assets of an account should not contain any tokens with zero balance)
Some tokens revert for zero value transfers (see https://github.com/d-xo/weird-erc20)
3.) Paused tokens
4.) Upgradeable tokens that changed the implementation.

## Impact
See above, an account cannot be liquidated. In certain conditions, this might even be triggerable by the user. For instance, a user could try to get on the USDC blacklist to avoid liquidations.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/core/AccountManager.sol#L384
https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/core/Account.sol#L166

## Tool used

Manual Review

## Recommendation
Catch reversions for the transfer and skip this asset (but it could be kept in the assets list to allow retries later on).

## Sentiment Team
Fixed as recommended. PR [here](https://github.com/sentimentxyz/protocol/pull/231).

## Lead Senior Watson
Confirmed fix. 
