hansfriese
# `AccountManager.settle()` might revert with insufficient funds.

## Summary
`AccountManager.settle()` might revert with insufficient funds.


## Vulnerability Detail
https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L324


## Impact
`AccountManager.settle()` might revert with insufficient funds.

Currently it tries to repay all amount of loans when the balance is positive but it will revert when the repaying amount is greater than current balance.


## Proof of Concept
As we can see [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L324), it tries to repay all debt.

But it will revert when the repaying amount is greater than balance [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L236).


## Tool used
Manual Review


## Recommendation
I think we can modify `repay()` to use the minimum value of inputted value `amt` and `borrowBalance` like below.

Then we can pass current balance in the `settle()` function.

```
function repay(address account, address token, uint amt)
    public
    onlyOwner(account)
{
    ILToken LToken = ILToken(registry.LTokenFor(token));
    if (address(LToken) == address(0))
        revert Errors.LTokenUnavailable();
    LToken.updateState();

    uint borrowBalance = LToken.getBorrowBalance(account); //++++++++++++++++++++
    if(amt > borrowBalance) 
        amt = borrowBalance;

    account.withdraw(address(LToken), token, amt);
    if (LToken.collectFrom(account, amt))
        IAccount(account).removeBorrow(token);
    if (IERC20(token).balanceOf(account) == 0)
        IAccount(account).removeAsset(token);
    emit Repay(account, msg.sender, token, amt);
}

function settle(address account) external onlyOwner(account) {
    address[] memory borrows = IAccount(account).getBorrows();
    for (uint i; i < borrows.length; i++) {
        uint balance;
        if (borrows[i] == address(0)) balance = account.balance;
        else balance = borrows[i].balanceOf(account);
        if ( balance > 0 ) repay(account, borrows[i], balance); //++++++++++++++++++++++++
    }
}
```