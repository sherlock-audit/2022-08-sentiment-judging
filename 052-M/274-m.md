WATCHPUG
# Reusing a recently closed account in `openAccount()` can result in unexpected allowances to previous spenders

## Summary

`openAccount()` will reuse the recently closed account if there is one. This is unexpected and can be dangerous if one of the previously approved spenders is malicious/compromised.

## Vulnerability Detail

When a previously approved spender is malicious/compromised due to vulnerability regarding allowance management, which can and already happened for a few reputable protocols, such as [DYDX](https://dydx.exchange/blog/deposit-proxy-post-mortem), [Anyswap](https://medium.com/multichainorg/anyswap-multichain-router-v3-exploit-statement-6833f1b7e6fb), [Sorbet Finance](https://medium.com/gelato-network/sorbet-finance-vulnerability-post-mortem-6f8fba78f109), etc.

Once learned about the vulnerability, the user may choose to close the account to avoid potential risks. (They may not be able to `approve` 0, if the admin as since removed the `controller.controllerFor(spender)`)

The user then tries to open a new account with `openAccount()` and move their funds to the new account, which is expected to open a brand new account, as it is unconventional that `openAccount()` will revive a recently closed account.

Once the funds being put into the account, the allowances to the previous spenders still exist, and attacker may steal the funds through the malicious/compromised spender.

## Impact

Users who reopens a closed account with allowance to malicious/compromised `target` may lose funds.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L90-L105

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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L113-L123

```solidity
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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L266-L278

```solidity
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

## Tool used

Manual Review

## Recommendation

1. Consider not reusing the closed account since it may also cause other unexpected risks from the previous states.

2. Consider adding a new function to reset allowances to 0 regardless if the spender is whitelisted (`address(controller.controllerFor(spender)) != address(0)`), furthermore, this function can be called not only by the account owner but also the system admin. This can be used to safeguard the users when one of the integrated protocols is exploited.