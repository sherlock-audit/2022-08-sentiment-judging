WATCHPUG
# Turning `isCollateralAllowed[token]` from `true` to `false` with `toggleCollateralStatus()` won't disallow the token to be continued used as collateral

## Summary

Admin can call `toggleCollateralStatus()` to turn one token's `isCollateralAllowed` from `ture` to `false`, but this can not prevent new borrows collateralized with the `token`, nor existing borrows to continue being collateralized with the `token`.

## Vulnerability Detail

1. Turning `isCollateralAllowed[token]` to `false` can not prevent new borrows collateralized with the `token`

When the admin found that one asset was not suitable to continue to be used as collateral for new loans, they called `toggleCollateralStatus()` and update `isCollateralAllowed[token]` from `true` to `false`.

The expected behavior is that new loans can not be borrowed with this asset.

The actual result is that users who already have such collateral in the assets list can still transfer directly through `IERC20(token).transfer()` and borrow more with the asset, as the collateral token is in the assets list.

2. Turning `isCollateralAllowed[token]` to `false` still allows existing loans collateralized with the `token` to continue being collateralized with the `token`

When the admin found that one asset was toxic, and should not be used as collateral for both existing and new loans. Instead of turning `isCollateralAllowed[token]` from `true` to `false`, it's better to change the oracle and return `0` for such an asset.

However, this should be carried out with extra caution, as this can immediately make accounts collateralized with the asset becomes liquidatable.

The current implementation is merely a flag to determine whether an asset can be ADDED as a NEW collateral.

## Impact

We believe this is more than a misnomer, but a real vulnerability that can cause material damage to the protocol when one of the scenarios occurs.

To exploit this, a sophisticated user can deposit and maintain at least 1 Wei for each of the whitelisted collateral assets to obtain the rights of using these assets as their collateral assets.

This means that whether these assets are prohibited from being used as collateral in the future, the collateral value will be calculated in `totalBalance()`.

This leads to that if one of the assets is no longer suitable or safe to be used as collateral, the user can still continue using these assets as collateral for existing and new loans, which can further accrue risks and bad debts to the protocol and liquidity provider of the LTokens.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L162-L173

```solidity
function deposit(address account, address token, uint amt)
    external
    whenNotPaused
    onlyOwner(account)
{
    if (!isCollateralAllowed[token])
        revert Errors.CollateralTypeRestricted();
    if (IAccount(account).hasAsset(token) == false)
        IAccount(account).addAsset(token);
    token.safeTransferFrom(msg.sender, account, amt);
}
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L395-L397

```solidity
function toggleCollateralStatus(address token) external adminOnly {
    isCollateralAllowed[token] = !isCollateralAllowed[token];
}
```

https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/uniswap/UniV2Controller.sol#L226-L245

```solidity
function swapErc20ForErc20(bytes calldata data)
    internal
    view
    returns (bool, address[] memory, address[] memory)
{
    (,, address[] memory path,,)
            = abi.decode(data, (uint, uint, address[], address, uint));

    address[] memory tokensOut = new address[](1);
    tokensOut[0] = path[0];

    address[] memory tokensIn = new address[](1);
    tokensIn[0] = path[path.length - 1];

    return(
        controller.isTokenAllowed(tokensIn[0]),
        tokensIn,
        tokensOut
    );
}
```

https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/core/ControllerFacade.sol#L65-L67

```solidity
function toggleTokenAllowance(address token) external adminOnly {
    isTokenAllowed[token] = !isTokenAllowed[token];
}
```

## Tool used

Manual Review

## Recommendation

Consider introducing a list of deprecated assets, when `RiskEngine.sol#isAccountHealthy()` calculating accounts balance the assets in that list come with a discount.

An asset can be added to the list and gradually increase the discount over a period of time until the asset is no longer considered as collateral as the discount is 100% or the price is 0.