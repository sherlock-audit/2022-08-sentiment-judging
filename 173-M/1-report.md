hansfriese
# liquidators couldn't liquidate other's account because of gas limit.

## Summary
liquidators couldn't liquidate other's account because of gas limit.


## Vulnerability Detail
https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L250


## Impact
When users liquidate other's account when it's unhealthy, they should repay all debts and receive all assets from the accounts within 1 external call.

It's a kind of unbounded loop and it might revert because of gas limit.

So a borrower can make others can't liquidate his account by borrowing and containing many tokens inside his account.


## Proof of Concept
First, a liquidator should repay debts and receive all assets inside one function [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L367).

This function might revert with the below scenario.

- A borrower deposited a certain amount of allowed collateral to his account.
- He borrowed all of the possible LTokens as he can.
- He swapped the tokens to increase the number of assets as much as possible using [exec()](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L290).
- When he uses the controllers, most of them have the list of allowed tokens [like this](https://github.com/sentimentxyz/controller/tree/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV2Controller.sol#L77).
- But others don't have such logic [here](https://github.com/sentimentxyz/controller/tree/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/yearn/YearnController.sol#L46) and [here](https://github.com/sentimentxyz/controller/tree/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/curve/CurveCryptoSwapController.sol#L174).
- Even if the borrower must use only allowed tokens during swapping, there is no upper limit so we don't know how many tokens he can have.
- So after the account is unhealthy, others can't liquidate his account because of the gas limit.
- When he decides to settle his account, he can reduce the available tokens using `exec()` and `repay()`.


## Tool used
Manual Review


## Recommendation
I think it would be good to add a limit that users can borrow and own inside the account.

For example, we can limit each user can have 5 borrowing tokens and 10 assets at most.