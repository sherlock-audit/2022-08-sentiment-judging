xiaoming90
# Division Rounding Cause Loss Of Ether During Redemption

## Summary

Division rounding errors can lead to a loss of Ether for users who attempt to redeem their shares from the LEther vault.

## Vulnerability Detail

Based on the `previewRedeem` and `convertToAssets` functions, it is possible that a user redeems/burns their shares but does not get back any assets in return due to a rounding error as `mulDivDown` function is used. This will occur when `shares * totalAsset` is smaller than `supply`.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/utils/ERC4626.sol#L154

```solidity
function previewRedeem(uint256 shares) public view virtual returns (uint256) {
    return convertToAssets(shares);
}
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/utils/ERC4626.sol#L132

```solidity
function convertToAssets(uint256 shares) public view virtual returns (uint256) {
    uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

    return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
}
```

The development team is already aware of this issue as they have implemented an additional validation check ` require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");` within the `redeem` function of `ERC4626` contract to ensure that the amount of assets returned from the `previewRedeem` function is not zero.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/utils/ERC4626.sol#L97

```solidity
function redeem(
    uint256 shares,
    address receiver,
    address owner
) public virtual returns (uint256 assets) {
    if (msg.sender != owner) {
        uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.

        if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
    }

    // Check for rounding error since we round down in previewRedeem.
    require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");

    beforeWithdraw(assets, shares);

    _burn(owner, shares);

    emit Withdraw(msg.sender, receiver, owner, assets, shares);

    asset.safeTransfer(receiver, assets);
}
```

However, the `redeemETH` function within the `LEther` contract did not implement the additional validation check to ensure that the amount of assets returned from the `previewRedeem` function is not zero. Thus, it is possible that the user calls the `redeemEth` function to redeem/burn their shares but does not get back any Ether in return.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LEther.sol#L47

```solidity
/**
    @notice Unwraps Eth and transfers it to the caller
        Amount of Eth transferred will be the total underlying assets that
        are represented by the shares
    @dev Emits Withdraw(caller, receiver, owner, assets, shares);
    @param shares Amount of shares to redeem
*/
function redeemEth(uint shares) external {
    uint assets = previewRedeem(shares);
    _burn(msg.sender, shares);
    emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
    IWETH(address(asset)).withdraw(assets);
    msg.sender.safeTransferEth(assets);
}
```

## Impact

Loss of Ether for the user who attempts to redeem their shares from the LEther vault.

## Recommendation

Implement additional check within the `redeemEth` function to ensure that the amount of assets returned from the `previewRedeem` function is not zero

```diff
function redeemEth(uint shares) external {
    uint assets = previewRedeem(shares);
+   require(assets != 0, "ZERO_ASSETS");
    _burn(msg.sender, shares);
    emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
    IWETH(address(asset)).withdraw(assets);
    msg.sender.safeTransferEth(assets);
}
```