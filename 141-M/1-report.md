berndartmueller
# Account liquidation can fail in certain situations

## Summary

Under certain conditions, unhealthy accounts cannot be liquidated and therefore add risk to the protocol of accumulating bad debt.

## Vulnerability Detail

Unhealthy accounts are subject to liquidation by liquidators who repay borrowed assets. Repaying borrowed assets and sweeping leftover account collateral happens within the same transaction. The `Account.sweepTo` function transfers all registered `assets` to a specified address within a loop.

However, token transfers can fail due to various reasons:

- Account address got ERC-20 token funds frozen/paused
- ERC-20 token contract is paused in general
- ...

Reverting token transfers within the `Account.sweepTo` function will cause sweeping and the account liquidation to fail. Hence, unhealthy accounts which cannot be liquidated are a risk to the protocol and will add bad debt.

## Impact

Unhealthy accounts can not be liquidated and will cause the Sentiment protocol to incur bad debt.

## Code Snippet

[protocol/src/core/Account.sol#L163-L174](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L163-L174)

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

`AccountManager.liquidate` calls `AccountManager._liquidate(account)`.

[protocol/src/core/AccountManager.sol#L253](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L253)

```solidity
function liquidate(address account) external {
    if (riskEngine.isAccountHealthy(account))
        revert Errors.AccountNotLiquidatable();
    _liquidate(account);
    emit AccountLiquidated(account, registry.ownerFor(account));
}
```

`AccountManager._liquidate` calls `account.sweepTo(msg.sender);` to sweep all assets.

[protocol/src/core/AccountManager.sol#L384](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L384)

```solidity
function _liquidate(address _account) internal {
    [..]

    account.sweepTo(msg.sender);
}
```

However, as explained above, if certain asset token transfers revert, sweeping will revert and so will the account liquidation.

## Tools Used

Manual review

## Recommendation

Consider splitting the liquidation of accounts into two separate parts:

1. Liquidate an unhealthy account by repaying borrowed amounts
2. Liquidator can sweep assets via a separate callable function by providing appropriate token addresses
