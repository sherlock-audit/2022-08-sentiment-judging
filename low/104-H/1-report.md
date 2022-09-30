kirk-baird
# HIGH: Unbounded Loops May Prevent Liquidation

## Summary

In `Account.sol` there's two unbounded arrays which iterate over `assets` and `borrows` when calling `liquidate()`. If these arrays become too large they may consume more gas than exists in the block gas limit to iterate over. 

## Vulnerability Detail

The function `AccountManager.liquidate()` requires iterating over two arrays controlled by the user. These arrays are `Account.borrows` and `Account.assets`.
If an attacker were to continue to call `deposit()`, `borrow()` and `exec()` to increase the size of these arrays then they may increase to a point where iterating over them will require more gas than exists in the block gas limit.

The function `liquidate()` would not be able to be called by any user for this `account`.

## Impact

Being unable to liquidate users results in the same impact as described in #1 . That is a malicious user may going into a negative position which cannot be liquidated and therefore the lenders lose funds.

Since users are able to `repay()` a `borrows` without iterating over all the `assets` and `borrows` they may decrease the size of this list if their position improves. They would be to repay all debts then withdraw all assets.
If the position does not improve they simply leave all `assets` and `borrows` preventing liquidation and causing the lenders to incur loss.

## Code Snippet

There are multiple occurrences of these loops.

`RiskEngine.sol` called from  `isAccountHealthy()` iterate over both `Account.borrows` and `Account.assets`
```solidity
    function _getBalance(address account) internal view returns (uint) {
        address[] memory assets = IAccount(account).getAssets();
        uint assetsLen = assets.length;
        uint totalBalance;
        for(uint i; i < assetsLen; ++i) {
            totalBalance += _valueInWei(
                assets[i],
                IERC20(assets[i]).balanceOf(account)
            );
        }
        return totalBalance + account.balance;
    }

    function _getBorrows(address account) internal view returns (uint) {
        if (IAccount(account).hasNoDebt()) return 0;
        address[] memory borrows = IAccount(account).getBorrows();
        uint borrowsLen = borrows.length;
        uint totalBorrows;
        for(uint i; i < borrowsLen; ++i) {
            address LTokenAddr = registry.LTokenFor(borrows[i]);
            totalBorrows += _valueInWei(
                borrows[i],
                ILToken(LTokenAddr).getBorrowBalance(account)
            );
        }
        return totalBorrows;
    }
```

`AccountManager._liquidate()` which iterates over `Account.borrows`
```solidity
    function _liquidate(address _account) internal {
        IAccount account = IAccount(_account);
        address[] memory accountBorrows = account.getBorrows();
        uint borrowLen = accountBorrows.length;

        ILToken LToken;
        uint amt;

        for(uint i; i < borrowLen; ++i) {
            address token = accountBorrows[i];
            LToken = ILToken(registry.LTokenFor(token));
            LToken.updateState();
            amt = LToken.getBorrowBalance(_account);
            token.safeTransferFrom(msg.sender, address(LToken), amt);
            LToken.collectFrom(_account, amt);
            account.removeBorrow(token);
        }
        account.sweepTo(msg.sender);
    }
```

Finally, `Account.sweepTo()` will iterate over `Account.assets`
```solidity
function sweepTo(address toAddress) external accountManagerOnly {
        uint assetsLen = assets.length;
        for(uint i; i < assetsLen; ++i) {
            assets[i].safeTransfer(
                toAddress,
                assets[i].balanceOf(address(this))
            );
            hasAsset[assets[i]] = false;
        }
        delete assets;
        toAddress.safeTransferEth(address(this).balance);
    }
```

## Tool used

Manual Review

## Recommendation

To avoid this issue consider limiting the total number of `assets` and `borrows` an account may have such calling `liquidate()` (or any other function) will not breach block gas limits.
