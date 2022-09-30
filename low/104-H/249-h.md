IllIllI
# Liquidations may be impossible when an Account holds too many assets

## Summary
There is no limit to the number of assets an account may hold, and this can lead to issues with gas limits

## Vulnerability Detail
Liquidation attempts to liquidate all assets in an account, and there may be so many assets that the liquidation transaction requires more gas than the block gas limit, causing it to always revert

## Impact
A user that should be liquidated cannot have their position liquidated, meaning the protocol has to cover the bad debt that may grow forever

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/core/Account.sol#L163-L174

## Tool used

Manual Review

## Recommendation
Allow subsets of the asset to be liquidated, rather than requiring all to be liquidated at once