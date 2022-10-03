xiaoming90
# New Account Inherits Past Approvals

## Summary

When a user requests a new account, an old account with existing approvals attached to it might be issued to the user.

## Vulnerability Detail

The owner of the Sentiment account can approve a spender to send a given amount of tokens from their Sentiment account.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L266

```solidity
/**
    @notice Gives a spender approval to spend a given amount of token from
        the account
    @dev Spender must have a controller in controller facade
    @param account Address of account
    @param token Address of token
    @param spender Address of spender
    @param amt Amount of token
*/
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

When the Sentiment account is closed, it is added to the `inactiveAccountsOf` list. However, it was observed that the existing approvals on the closed Sentiment account are not being cleared and still remain on the Sentiment account address.

Assume that Alice owns a Sentiment account with an address of `0x12345`. Alice approves the Compound protocol to spend all the WETH tokens in her Sentiment account by setting the spend amount to max amount (`type(uint256).max`). Alice then decides to close her Sentiment account, and her Sentiment account (0x12345) is added to the `inactiveAccountsOf` list.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L113

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

A few months later, Alice decides to open a new Sentiment account again by calling the `openAccount` function. Users will expect a brand new Sentiment account with a clean slate to be issued to them after they open a new account. However, this might not be the case sometime. For instance, since Alice has closed a Sentiment account in the past, her old Sentiment account (0x12345) will be released from the `inactiveAccountsOf` list and issued to Alice instead of creating a Sentiment account with a new address. This is mainly for gas-saving purposes.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L90

```solidity
/**
    @notice Opens a new account for a user
    @dev Creates a new account if there are no inactive accounts otherwise
        reuses an already inactive account
        Emits AccountAssigned(account, owner) event
    @param owner Owner of the newly opened account
*/
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

## Impact

In the above example, by getting back her old Sentiment account (0x12345), Alice also "inherited" the past approval she had made in the past, which is to allow Compound to spend all the WETH within the account. However, since Alice has already closed her old account, she will expect all the states of the old account to be erased. Most importantly, since Alice has instructed Sentiment to open a new account, she will expect to be given a new account with a clean slate without any past approvals attached to it.

Next, assume the following scenario:

1) Alice transferred 1m WETH to the new account
2) Compound is compromised, and the attacker decided to pull/steal all WETH from those addresses that allow Compound to spend them
3) 1m WETH in Alice's account (0x12345) are transferred to the attacker's address
4) Alice will be surprised as she did not approve Compound to spend all her WETH in her new account.
5) Affected users (including Alice) found out that Sentiment did not properly clear the approvals of closed accounts, put the blame on Sentiment, and demanded compensation. This leads to a loss of reputation and funds for Sentiment.

## Recommendation

Consider implementing one of the following mitigations:

1. Update the implementation to deploy a new Sentiment contract whenever a user opens an account. Consider minimal proxy that creates a cheap clone if gas efficiency is a concern. Do not reuse old Sentiment accounts that have been closed. (Recommended Solution)
2. Whenever a user calls `approve` on their Sentiment account, record the approval details in a listing. When the user closes their Sentiment account, loop through the approval details in the listing and reset the approval to zero.