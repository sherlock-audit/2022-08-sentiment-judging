cccz
# AccountManager: Anyone can call openAccount on any address, which may cause the user to fail in the loop of the accountsOwnedBy() due to exceeding the block gas limit.

## Summary
AccountManager: Anyone can call openAccount on any address, which may cause the user to fail in the loop of the accountsOwnedBy() due to exceeding the block gas limit.
## Vulnerability Detail
In the AccountManager contract, anyone can call openAccount() on any address to increase the number of Accounts at that address.
Later, if a user uses the accountsOwnedBy function of the Registry contract to look up their Account, the function may fail due to too much gas consumed by too many Accounts that match the ownerFor[accounts[i]] == user condition. When the consumed gas exceeds the block gas limit, the function will not run successfully.

## Impact
The user may not be able to look up his Account by the accountsOwnedBy() function.
## Code Snippet
```
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
...
    function addAccount(address account, address owner)
        external
        accountManagerOnly
    {
        ownerFor[account] = owner;
        accounts.push(account);
        emit AccountCreated(account, owner);
    }
...
    function accountsOwnedBy(address user)
        external
        view
        returns (address[] memory userAccounts)
    {
        userAccounts = new address[](accounts.length);
        uint index;
        for (uint i; i < accounts.length; i++) {
            if (ownerFor[accounts[i]] == user) {
                userAccounts[index] = accounts[i];
                index++;
            }
        }
        assembly { mstore(userAccounts, index) }
    }
```
## Tool used

Manual Review

## Recommendation

Consider allowing the user to call the openAccount function only for himself