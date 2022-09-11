grhkm
# AccountManager.liquidate() allows any user to steal any additional token or ETH held in contract 

## Summary
Any user who liquidates an account will be transferred all tokens or ETH that were sent directly to the account contract.

## Vulnerability Detail
When a user calls AccountManager.liquidate() , they receive all token assets and ETH held in the account contract which also includes those which were sent directly to the contract so long the account is unhealthy.

This means a user could possibly target accounts that holds extra ETH and/or tokens and they satisfy the account unhealthy condition. 

## Impact
Loss of contract funds

## Code Snippet
**Account Sweep**
```
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
Above transfers all assets and eth held in the account contract which may not match the total borrows by the account, allowing any user that calls liquidates and pays the token amount to LToken contract, to walk away with some profits in both eth and erc20 token assets.

## Tool used

Manual Review

## Recommendation
The sweepTo() function's logic should be that which sends the amount of assets liquidated by the user and not all tokens held by account contract.
