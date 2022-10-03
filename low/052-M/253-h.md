IllIllI
# Accounts are never un-approved for a spender when re-used

## Summary
Accounts are allowed to `approve()` spenders if there is a controller for a spender

## Vulnerability Detail
Once approved, there is nothing that unapproves a spender when the account is closed

## Impact
A spender may notice that the account is being re-used by a whale, and will be able to transfer out the whale's funds

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/core/AccountManager.sol#L266-L278

## Tool used

Manual Review

## Recommendation
Set approval to zero during account closure