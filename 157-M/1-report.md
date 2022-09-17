Tutturu
# Users can withdraw borrowed assets

## Summary
The protocol allows borrowed assets to be withdrawn as long as collateralValue - valueOfWithdrawnTokens / accountBorrowsValue is > 1.2e18

## Vulnerability Detail
 ```AccountManager - withdraw()``` as well ```riskEngine.isWithdrawAllowed()``` never check if the asset is borrowed or deposited as collateral. This allows users to withdraw their borrowed assets.

## Impact
While this does not lead to an immediate loss of funds this behaviour can have unexpected consequences, and it breaks one of the core protocol assumptions defined in the docs:

*Since the account is a proxy contract which is separate from the borrower's EOA, **the borrower never has custody of the loaned assets** i.e. neither the deposited collateral nor the borrowed assets can be transferred out of the account.* -  [docs#Account](https://docs.sentiment.xyz/core-concepts/account)

Previous issues I've submitted demonstrate two possible ways to exploit this behaviour.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Disallow withdrawing borrowed assets.
