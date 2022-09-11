grhkm
# Interest is not collected until borrower choses to

https://github.com/sentimentxyz/protocol/blob/main/src/tokens/LToken.sol#L153
## [M] Interest is not collected until borrower choses to
### Problem
When LToken lends its underlying asset it does not have means to collect it until borrower decides to repay or gets liquidated.
If borrower has a lot of collateral, he can chose not to repay for any amount of time. 
LToken only increases record of how much borrower is owed, but interest is never paid until liquidation or repay.
Lenders may choose to withdraw available for LToken underlying asset and will be available to get only amount of asset that vault actally owns, so late withrawers may end up in a situation when vault does not own any underlying asset, but only has `borrows` record, which will be rapaid god knows when, maybe in years time.
### Proof of Concept
- Alice deposits to LToken vault 100 ETH worth of some asset
- Bob's account has 1k ETH  collateral 
- Bob borrows 50 ETH
- LToken vault has only 50 ETH worth of underlying asset
- Alice can withdraw only 50 ETH OR 
- Charlie deposits 50 ETH
- Alice withdraws 100 ETH
- Charlie can't withdraw anything, not even interest accrued, until Bob choses to repay or get's liquidated, which may take years.
I guess the LToken vault becomes temporarily insolvent 
### Mitigation
Make it possible to collect some actual value from Accounts after some time intervals.