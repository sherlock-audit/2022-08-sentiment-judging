Kumpa
# ```AccountManager.approve()``` could allow the spender to spend the token beyond the RiskThreshold

All functions that associate with the outflow of tokens have ```riskEngine.isAccountHealthy(account)``` to check the status of account if it is eligible with the exception of ```AccountManager.approve()```. This could arbitrary allow the spender to spend the approved token from the account beyond the threshold limit.

### Proof of Concept
1.The user has Balance = 100 while Borrow = 50. This makes collateral ratio to be 2

2.The user approve the spender of the account with the maximum amount of tokens in the account

3.The spender transfer all the token in the account, casuing collateral ratio to be 0 which is below ```balanceToBorrowThreshold``` or 1.2

### Mitigation
Add a check on the amount that can be spent to see if the account status is still healthy even after that amount of token is spent.

## Comment

Misses specific details regarding the specific controller used.