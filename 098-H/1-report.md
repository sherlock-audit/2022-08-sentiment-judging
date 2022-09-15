yixxas
# Unlimited borrow can be made starting with a small amount of token

## Summary
Any user can borrow a token as long as they keep a healthy balance of tokens owned to the borrowed amount. However, calculation of balance of a user is inclusive of the borrowed amount, hence it can be abused, where loans can be used to make even more loans.

## Vulnerability Detail

We see in the function `borrow()`, the only check done on a borrower is `!riskEngine.isBorrowAllowed()`.
```solidity
    function borrow(address account, address token, uint amt)
        external
        whenNotPaused
        onlyOwner(account)
    {   
        if (registry.LTokenFor(token) == address(0))
            revert Errors.LTokenUnavailable();
        if (!riskEngine.isBorrowAllowed(account, token, amt))
            revert Errors.RiskThresholdBreached();
        if (IAccount(account).hasAsset(token) == false)
            IAccount(account).addAsset(token);
        if (ILToken(registry.LTokenFor(token)).lendTo(account, amt))
            IAccount(account).addBorrow(token);
        emit Borrow(account, msg.sender, token, amt);
    } 
```
`isBorrowAllowed()` checks if ` (_getBalance(account) + borrowValue) / (_getBorrows(account) + borrowValue)` > some threshold. ( currently set at 1.2 ) So, an account with 100 tokens can make a borrow of 80 tokens, since `100/80 = 1.25 > 1.2`.
```solidity
    function isBorrowAllowed(
        address account,
        address token,
        uint amt 
    )   
        external
        view
        returns (bool)
    {   
        uint borrowValue = _valueInWei(token, amt);
        return _isAccountHealthy(
            _getBalance(account) + borrowValue,
            _getBorrows(account) + borrowValue
        );  
    } 
```
In the `lendTo()` function, which is called by `borrow()`, `asset.safeTransfer()` is called. This updates the borrower's balance to be inclusive of the borrowed amount.
```solidity
    function lendTo(address account, uint amt)
        external
        whenNotPaused
        accountManagerOnly
        returns (bool isFirstBorrow)
    {   
        ...
        asset.safeTransfer(account, amt);
        return isFirstBorrow;
    }
```
Hence an account starting with 100 tokens, can first borrow 80 tokens.
`balanceOf(account)` = 180
`borrowsOf[account]` = 80

With the borrowed amount, the account can, for example make another borrow of 80 tokens, since `(180 + 80) / (80 + 80) = 1.625 > 1.2`. This borrow loop can go on forever, draining all the tokens from the contract by starting with a small amount of tokens.

## Impact

This essentially means that all funds can be stolen by continuously borrowing, since an attacker would only need to invest a small amount of token. The account will also be safe from liquidation, since `balanceOf(account)` includes borrowed amount.

## Tool used

Manual Review

## Recommendation

`balanceOf(account)` should only take into account the base amount that an account has when checking if an account is able to borrow. An account with 100 tokens should only be able to borrow `100/1.2` tokens, and not be able to reuse the borrowed amount to make more borrows.

