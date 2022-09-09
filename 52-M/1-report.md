minera
# Token approval not removed when account is closed in AccountManager.sol

## Summary

Token approval allowances are not removed when the account is closed in AccountManager.sol

then the account is marked as inactive and can be reactivated for another owner.

The accounts could be at risk of having their token balances spent by a malicious third party if a previous owner of the account
approved that controller as a spender.

## Vulnerability Detail

In AccountManager.sol, an account owner can approve spender using the function approve.

```
    function approve(
        address account,
        address token,
        address spender,
        uint amt
    )
        external
        onlyOwner(account)
    {
        if(address(controller.controllerFor(spender)) == address(0))
            revert Errors.FunctionCallRestricted();
        account.safeApprove(token, spender, amt);
    }
```

but when closeAccount is closed and marked as inactive but the token approval is not removed.

```

    /**
        @notice Closes a specified account for a user
        @dev Account can only be closed when the account has no debt
            Emits AccountClosed(account, owner) event
        @param _account Address of account to be closed
    */
    function closeAccount(address _account) public onlyOwner(_account) {
        IAccount account = IAccount(_account);
        if (account.activationBlock() == block.number)
            revert Errors.AccountDeactivationFailure();
        if (!account.hasNoDebt()) revert Errors.OutstandingDebt();
        account.deactivate();
        registry.closeAccount(_account);
        inactiveAccountsOf[msg.sender].push(_account);
        account.sweepTo(msg.sender);
        emit AccountClosed(_account, msg.sender);
    }
```

when a new user create an account, the protocol will first reactivate an inactive account,
with existing token approval and the new owner of the account will inherit those approvals.

## Impact

The accounts could be at risk of having their token balances spent by a malicious third party if a previous owner of the account
approved that controller as a spender.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Remove and revoke token approval allowance when account is closed.

or remove the account and create a new account every time when the owner wants to create account.
