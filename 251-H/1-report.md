IllIllI
# No method to repay debts of toxic assets

## Summary
An asset may become toxic, e.g. due to being added to an OFAC list a-la the recent Whirlpool action. 

## Vulnerability Detail
In such cases, there may be nobody that wishes to take on the toxic asset, and therefore the borrower's position may never be liquidated, leading to bad debt that continues to grow

## Impact
Debt that is never repaid will lead to the protocol becoming insolvent

## Code Snippet
Liquidation is all-or-nothing, meaning the liquidator will get the toxic asset:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/2e25699040ed87a9af62f2b637eafcc6b0b59cc5/protocol/src/core/Account.sol#L163-L174


https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/2e25699040ed87a9af62f2b637eafcc6b0b59cc5/protocol/src/core/AccountManager.sol#L367-L385

## Tool used

Manual Review

## Recommendation
Have an insurance fund that can be used as the liquidator of last resort, controlled by a DAO, and let liquidators liquidate only portions of the debt