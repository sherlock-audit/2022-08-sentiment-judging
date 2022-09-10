cergyk
# Closed Accounts of Victim can be reopened by any user

## Summary
It was observed that any user can open account for any other user. This becomes a problem as shown in poc

## Vulnerability Detail

1. User X decides to close his account A
2. This adds a new entry to [inactiveAccountsOf](https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L120) as shown below

```
inactiveAccountsOf[X].push(A);
```

3. Attacker uses [openAccount](https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L90) function and passes owner as User X

```
function openAccount(address owner) external whenNotPaused {
        ...
        if (length == 0) {
            ...
        } else {
            account = inactiveAccountsOf[owner][length - 1];
            inactiveAccountsOf[owner].pop();
            registry.updateAccount(account, owner);
        }
        ...
    }
```
4. So this makes use of inactiveAccountsOf[X] which is Account A from Step 2
5.  This means Account A from User X again get reactivated even though Victim never wanted the same

## Impact
Victim closed account inactiveAccountsOf, can be forcefully opened by Attacker even though Victim want to cut off all ties with the inactive account

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L90

## Tool used
Manual Review

## Recommendation
Revise openAccount as shown below

```
function openAccount() external whenNotPaused {
        address owner=msg.sender;
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