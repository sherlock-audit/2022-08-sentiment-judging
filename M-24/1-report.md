Lambda
# AccountManager: Fee-On-Transfer tokens not supported

## Summary
Fee-on-transfer tokens lead to problems in `AccountManager`.

## Vulnerability Detail
`AccountManager` assumes when liquidating / repaying debt (see code snippets) that the transferred amount equals to the received amount, which is not the case for fee-on-transfer tokens.

## Impact
When a fee-on-transfer token is used, the debt is reduced by the whole amount, although only part of it is actually transferred to the `LToken`. This destroys the accounting of the `LToken` and is bad for the token holders.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/AccountManager.sol#L380
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/AccountManager.sol#L236

## Tool used

Manual Review

## Recommendation
The actual amount that was transferred could be checked. However, not supporting fee-on-transfer tokens is also completely fine in my opinion. Maybe that is already intended, but I did not find anything and wanted to make sure that you are aware of the issues with these tokens.