rajatbeladiya
# Anyone can active accounts of inactive accounts

## Summary
Attacker can active account from anyone's inactive accounts.

## Vulnerability Detail
1. Alice opens account using `openAccount()`
2. Alice closes account using `closeAccount()`
3. Attacker can activate account using `openAccount()` with `Alice's` address.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-rajatbeladiya/blob/4635dc8ab97aff9f4e3ce29b73f98c268eae4533/protocol/src/core/AccountManager.sol#L90-L123

## Tool used
Manual Review

## Recommendation
Add check for the owner of account can update the account