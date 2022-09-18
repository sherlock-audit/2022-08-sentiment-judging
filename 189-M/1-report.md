rbserver
# User cannot liquidate account when calling `sweepTo` function reverts

## Summary
When calling the `sweepTo` function reverts, calling the `_liquidate` function also reverts, which causes the user to not be able to liquidate the liquidable account.

## Vulnerability Detail
As shown in the Code Snippet section, calling the `sweepTo` function, which is executed when calling the `_liquidate` function, would iterate over each asset in the `assets` array.

When one or more of the following events occur, it is possible that the block gas limit is reached before completing the transaction that includes the `sweepTo` function call.
1. Number of assets becomes very large.
2. Some assets' `transfer` functions cost much gas.
3. The gas costs for the needed operations for transferring the assets increase in the future.

Also, if calling the `transfer` function of one of the assets reverts, then calling `sweepTo` reverts. For example, the admin of an asset, such as `USDC` or `USDT`, can block the account and/or the receiver address(es) from sending or receiving the asset, which causes the asset's `transfer` function call to revert.

For both cases mentioned above, calling `sweepTo` would revert, which means that calling `_liquidate` would revert as well.

## Impact
For the described scenarios mentioned in the Vulnerability Detail section, the DOS can last for a long time, even forever such as if the admin of the asset never removes the affected address from the blocklist after blocking it. Because calling `_liquidate` reverts, users cannot liquidate the liquidable account while the account owner cannot withdraw assets from the account that is already liquidable. As a result, the relevent assets are locked in the account, and no one has access to these.

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L163-L174
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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L367-L385
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

## Tool used
Manual Review

## Recommendation
A function that supports partial liquidation can be added. When calling this function, the user can specify the assets from the liquidable account that they want to liquidate for, and these specified assets, instead of all assets of the account, would be iterated over to mitigate the described risk.