Sm4rty
# Unbounded Loop can lead to DOS due to out of gas.

## Summary
Unbounded Loop can lead to DOS due to out of gas.

## Vulnerability Detail:
As this array can grow quite large, the transaction’s gas cost could exceed the block gas limit and make it impossible to call this function at all (see @Audit):

## Impact
It can lead to DOS due to being out of gas and it will cause the transfer to revert.

## Code Snippet
accounts array is defined here. We can see that it is dynamic array:
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L29
```
    address[] public accounts;
```

Here addresses are pushed into accounts array: 
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L113-L120
```    function addAccount(address account, address owner)
        external
        accountManagerOnly
    {
        ownerFor[account] = owner;
        accounts.push(account);
        emit AccountCreated(account, owner);
    }
```
Here loop is unbounded, there is no upperbound, which can lead to out of gas situation.
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L183-L187
```
        for (uint i; i < accounts.length; i++) {
            if (ownerFor[accounts[i]] == user) {
                userAccounts[index] = accounts[i];
                index++;
            }
```


## Tool used
Manual Review

## Recommendations
Consider introducing a reasonable upper limit based on block gas limits and/or adding a remove method to remove elements in the array.