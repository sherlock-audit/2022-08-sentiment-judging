hyh
# AccountManager's repay called with the maximum amount might not close the account

## Summary

There is two rounding operations involved in the shares computations when repay() is called to close the account fully. Such logic results in the account not being closed in some cases as the operations do not always cancel each other. As a result in these cases the borrowing amount will remain positive, and account closure be still impossible.

## Vulnerability Detail

repay() called with `amt == type(uint256).max` might not close the corresponding borrowing position fully as `amt = LToken.getBorrowBalance(account)` means `shares = convertAssetToBorrowShares(convertBorrowSharesToAsset(borrowsOf[account]))`. But convertBorrowSharesToAsset() rounds down, while convertAssetToBorrowShares() rounds up, so the result can vary. I.e. one operation can change the number, while another can leave it as is, so, in general, resulting `shares != borrowsOf[account]`.

## Impact

A sweepTo() function retrieving all the account assets is only available in liquidation and account closure scenarios. I.e. the only way for the account owner to get back all the account holdings is to remove all the borrows so hasNoDebt() be `true` and the account can be closed.

This was the impact is that the owner repaying the last borrow and expecting to sweep all the collateral from the account will not be able to do so, i.e. it's a temporal fund freeze for the user.

## Code Snippet

repay() calls getBorrowBalance() to obtain the `amt` shares to withdraw when a user wants to close the position, calling repay() with `amt = type(uint256).max`:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/core/AccountManager.sol#L227-L242

getBorrowBalance() calls convertBorrowSharesToAsset() with `borrowsOf[account]`:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LToken.sol#L167-L174

convertAssetToBorrowShares uses mulDivUp(), convertBorrowSharesToAsset() uses mulDivDown():

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LToken.sol#L229-L237

This way `amt` getBorrowBalance() returns can be less than `borrowsOf[account]` and collectFrom() can yield `false`:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LToken.sol#L153-L165

So repay() will not remove `IAccount(account).removeBorrow(token)` and, if the debt was the last one, this will cause hasNoDebt() to be remain false and closeAccount() to revert:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/core/Account.sol#L135-L137

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/core/AccountManager.sol#L113-L121

## Recommendation

Consider introducing closure flag and using `borrowsOf[account]` without double conversion, for example:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/core/AccountManager.sol#L235-L238

```solidity
+       bool close;
-       if (amt == type(uint256).max) amt = LToken.getBorrowBalance(account);
+       if (amt == type(uint256).max) {amt = LToken.getBorrowBalance(account); close=true;}
        account.withdraw(address(LToken), token, amt);
+       LToken.collectFrom(account, amt, close);
-       if (LToken.collectFrom(account, amt))
+       if (close)
            IAccount(account).removeBorrow(token);
```

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/core/AccountManager.sol#L379-L382

```solidity
            amt = LToken.getBorrowBalance(_account);
            token.safeTransferFrom(msg.sender, address(LToken), amt);
-           LToken.collectFrom(_account, amt);
+           LToken.collectFrom(_account, amt, true);
            account.removeBorrow(token);
```

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LToken.sol#L153-L165

```solidity
-   function collectFrom(address account, uint amt)
+   function collectFrom(address account, uint amt, bool close)
        external
        accountManagerOnly
        returns (bool)
    {
-       uint borrowShares;
-       require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
+       uint borrowShares = close ? borrowsOf[account] : convertAssetToBorrowShares(amt);
+       require(borrowShares != 0, "ZERO_BORROW_SHARES");
        borrowsOf[account] -= borrowShares;
        totalBorrowShares -= borrowShares;

        borrows -= amt;
        return (borrowsOf[account] == 0);
    }
```