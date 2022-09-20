Czar102
# Liquidation DoS in case of a single collateral token failure

## Summary

If any of the tokens that may be used as collateral fail (for example, selfdestruct), change implementation to one that mistakenly doesn't fully comply with ERC20 or, for example, is temporarily paused (or is shut down, even with notice), while still being on the list of assets, it will DoS the liquidate function for an entire account.

## Vulnerability Detail

Note that the `sweepTo` function, and since the liquidation, won't finish execution (will revert) on any of the following:
- unbound gas usage of the `transfer` or `balanceOf` function
- if `assets[i]` is not a contract (anymore)
- if `transfer` or `balanceOf` calls revert, for any reason

## Impact

It is highly dangerous to leave such trust to all tokens that have ever been used as collaterals. Note that the failure of a single token that has ever been available causes the failure of the whole system.

DoS of liquidation causes the system not to be able to enforce keeping positions healthy, thus losing funds.

## Code Snippet

```js
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

Restrict gas usage, don't revert on call failure or if the address is not a contract.
