WATCHPUG
# Accounts with ETH loans can not be liquidated if LEther's underlying is set to `address(0)`

## Summary

Setting `address(0)` as LEther's `underlying` is allowed, and the logic in `AccountManager#settle()` and `RiskEngine#_valueInWei()` handles `address(0)` specially, which implies that `address(0)` can be an asset.

However, if LEther's underlying is set to `address(0)`, the accounts with ETH loans will become unable to be liquidated.

## Vulnerability Detail

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L318-L326

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L178-L188

Given that at `AccountManager.sol#L100` in `settle()` and `RiskEngine.sol#L186` in `_valueInWei()`, they both handled the case that the `asset == address(0)`, and in `Registry.sol#setLToken()`, `underlying == address(0)` is allowed:

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95-L105

We assume that `address(0)` can be set as the `underlying` of `LEther`.

In that case, when the user borrows native tokens, `address(0)` will be added to the user's assets and borrows list.

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L203-L217

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

This will later prevent the user from being liquidated because in `riskEngine.isAccountHealthy()`, it calls `_getBalance()` in the for loop of all the assets, which assumes all the assets complies with `IERC20`. Thus, the transaction will revert at L157 when calling `IERC20(address(0)).balanceOf(account)`.

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L250-L255

```solidity
function liquidate(address account) external {
    if (riskEngine.isAccountHealthy(account))
        revert Errors.AccountNotLiquidatable();
    _liquidate(account);
    emit AccountLiquidated(account, registry.ownerFor(account));
}
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L150-L161

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
```

## Impact

We noticed that in the deployment documentation, LEther is set to init with WETH as the `underlying`. Therefore, this should not be an issue if the system is being deployed correctly.

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/protocol/deployments/ArbiDeploymentFlow.md#L47-L53

```markdown=47
1. ETH
   1. Deploy LEther implementation
   2. Deploy Proxy(LEther)
   3. call init(WETH), "LEther", "LEth", IRegistry, reserveFactor)
   4. call Registry.setLToken(WETH, Proxy)
   5. call accountManager.toggleCollateralStatus(token)
   6. call Proxy.initDep()
```

But considering that setting `address(0)` as LEther's `underlying` is still plausible and the potential damage to the whole protocol is high (all the accounts with ETH loans can not be liquidated), we believe that this should be a medium severity issue.

## Code Snippet

## Tool used

Manual Review

## Recommendation

1. Consider removing the misleading logic in `AccountManager#settle()` and `RiskEngine#_valueInWei()` that handles `address(0)` as an asset;
2. Consider disallowing adding `address(0)` as `underlying` in `setLToken()`.