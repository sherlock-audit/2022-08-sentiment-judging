hansfriese
# `Account` contract might be locked when `underlying` = address(0).

## Summary
`Account` contract might be locked when `underlying` = address(0).


## Vulnerability Detail
https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95

https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L322


## Impact
Currently, it doesn't check the `underlying` of the LToken shouldn't be address(0).

If users borrow that LToken, all funds inside the `Account` contract will be locked.


## Proof of Concept
As we can see [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95), it doesn't check `underlying != address(0)`.

Also from [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L322), it looks like the borrowed token can be address(0).

So it looks like there can be a LToken of address(0) to borrow ETH.

But once such LToken is added, the below scenario would be possible.

- A user borrows address(0) using [borrow()](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L203).
- Then address(0) will be added to `Account.assets` [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L212-L213).
- Then [_getBalance()](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L150) will revert [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L157) and several functions like `isBorrowAllowed()` and `isWithdrawAllowed` will revert.
- `AccountManager.repay()` will revert also [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L239-L240).

So users can't withdraw the funds from `Account` contract forever.


## Tool used
Manual Review


## Recommendation
We should check `underlying != address(0)` in [Registry.setLToken()](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95).