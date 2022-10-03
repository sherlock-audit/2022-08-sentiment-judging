cccz
# Users can add a variety of collateral and borrowings to prevent themselves from being liquidated

## Summary
Users can add a variety of collateral and borrowings to prevent themselves from being liquidated
## Vulnerability Detail
In the _liquidate function of the AccountManager contract, all the borrowings of the liquidated person are iterated through and repaid by the liquidator. Then, in the sweepTo function, all the collateral of the liquidated person is iterated through and transferred to the liquidator.
Considering that the number of iterations increases when the liquidated person adds collateral and borrowings, when the _liquidate function exceeds the block gas limit due to unbounded iterations, liquidation will not be possible.
This should be a medium vulnerability considering that the contract will limit the collateral and borrowings, but if the contract integrates many assets, this could be a high vulnerability.
## Impact

Users can add a variety of collateral and borrowings to prevent themselves from being liquidated

## Code Snippet
```
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
...
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

Consider limiting the length of borrows/assets in the Account contract
