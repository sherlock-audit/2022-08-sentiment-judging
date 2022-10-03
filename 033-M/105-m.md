kirk-baird
# MED/HIGH: Blacklisting An Asset Prevents Liquidation

## Summary

If an `Account` is blacklisted for a token which is in the `assets` list it's impossible to be liquidated.

## Vulnerability Detail

Addresses may be blacklisted for certain tokens. For example USDC will blacklist accounts which have funds transferred to it from tornado cash. Once an address is blacklisted it will be impossible to transfer tokens. 

The result of blacklisting an `Account` which has an asset is that it's no longer possible to call `sweepTo()` since the transfer of the blacklisted asset will revert causing the entire transaction to revert.

Since the function `AccountManager.liquidate()` will call `Account.sweepTo()` to transfer all ERC20 tokens to the liquidator. If one of these tokens is blacklisted it will be impossible to call `liquidate()` to that account.

## Impact

The impact is similar to #1 and #2 . That is if it's not possible to call `liquidate()` then a user may take out a risky position which will cause a loss to lenders if this position goes negative. If the position goes positive the attacker may still repay the debts and withdraw any assets, other than the blacklisted asset, which may be a trivially small amount if blacklisted intentionally.

## Code Snippet

`AccountManager.sweepTo()` calls `assets[i].transfer()`.
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

Consider allowing the liquidator to only claim certain tokens. That is have `liquidate()` take an array of tokens to claim. Then performing `sweepToLimited(IERC20[] tokens)` only transfer the tokens requested by the liquidator.

This will prevent an attacker from being blacklisted with a very small amount of tokens (e.g. $1 worth) thus avoiding liquidations without incuring significant loss from having tokens locked.
