minhquanym
# Possible DOS because of unbounded gas consumption in some functions.

## Vulnerability Detail
- https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L159
- https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L176
- https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L163
- https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L185

Each block has a block gas limit and if a transaction consumes more than that, every call to that function will be reverted. In Sentiment, there are some places using loop through a storage array and do not have any mechanism to limit the loop. In case the array becomes too large, these function will be Denial-of-Service (DOS).

## Impact

In case `Registry.accounts` becomes too large, function `getAllAccounts()` cannot be used because it read whole `accounts` array from storage. Note that anyone can call `AccountManager.openAccount()` to add element to `accounts` array.
```solidity
function getAllAccounts() external view returns (address[] memory) {
    return accounts;
}
```

In case `Account.assets` becomes too large (it is possible if Sentiment allows too many tokens be collateral and borrow assets), then function `sweepTo()` can be DOS. It makes this Account cannot be liquidated. Function `removeAsset()` and `removeBorrows()` can be DOS as well, makes users unable to repay/withdraw.

## Proof of Concept

Attacker can call `openAccount()` for anyone infinite number of times. [Line 90](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L90)
```solidity
function openAccount(address owner) external whenNotPaused {
    if (owner == address(0)) revert Errors.ZeroAddress();
    address account;
    uint length = inactiveAccountsOf[owner].length;
    if (length == 0) {
        account = accountFactory.create(address(this));
        IAccount(account).init(address(this));
        registry.addAccount(account, owner);
    } else {
        account = inactiveAccountsOf[owner][length - 1];
        inactiveAccountsOf[owner].pop();
        registry.updateAccount(account, owner);
    }
    IAccount(account).activate();
    emit AccountAssigned(account, owner);
}
```

It called to `registry.addAccount()` and this function simply adds new account to its `accounts` list. [Line 113](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L113)
```solidity
function addAccount(address account, address owner)
    external
    accountManagerOnly
{
    ownerFor[account] = owner;
    accounts.push(account);
    emit AccountCreated(account, owner);
}
```

## Tool used

Manual Review

## Recommendation

Consider add a mechanism to limit the loop
1. Break it to multiple calls. For example `getAllAccountsInRange()` instead of `getAllAccounts()`
2. Make sure not too much tokens being whitelisted to be collateral