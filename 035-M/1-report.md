IllIllI
# Balances of rebasing tokens aren't properly tracked

## Summary
Rebasing tokens are tokens where `balanceOf()` returns larger amounts over time, due to the addition of interest to each account, or due to airdrops

## Vulnerability Detail
Sentiment doesn't properly track balance changes while rebasing tokens are in the borrower's account

## Impact
The lender will miss out on gains that should have accrued to them while the asset was lent out. While market-based price corrections may be able to handle interest that is accrued to everyone, market approaches won't work when only subsets of token addresses are given rewards, e.g. an airdrop based on a snapshot of activity that happened prior to the token being lent.

## Code Snippet
Lending tracks shares of the LToken:
https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/LToken.sol#L140-L143

But repayment assumes that shares are equal to the same amount, regardless of which address held them, which is not true for airdrops:
https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/LToken.sol#L160-L163

Rebasing tokens are supported, since Aave is a rebasing token:
https://github.com/sherlock-audit/2022-08-sentiment/blob/main/controller/src/aave/AaveEthController.sol#L28
## Tool used

Manual Review

## Recommendation
Adjust share amounts when the account balance doesn't match the share conversion calculation when taking into account gains made by the borrower

## Sentiment Team
We'll make sure to not interact with fee-on-transfer tokens. This can be ensured by the admins.

## Lead Senior Watson
Note: The admins should not add rebasing/fee-on-transfer tokens to any allowed lists.
