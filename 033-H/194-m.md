Chom
# If an asset is blocked from the transfer, using that asset as collateral will prevent liquidation

## Summary
If an asset is blocked from the transfer, using that asset as collateral will prevent liquidation

## Vulnerability Detail

_liquidate function calls sweepTo which transfers assets to the liquidator. But if safeTransfer always failed due to the transfer being blocked, liquidate function will always revert. Users won't be liquidated and cause a serious bad debt.

This may happen with USDC as they ban tornado cash. USDT can freeze cash too. It may happen anytime you can't know in advance.

## Impact
If that user uses an asset whose transfer is blocked as collateral and uses another asset that price is crashing but the transfer is not blocked as collateral too, that user can't be liquidated. This will cause a very serious bad debt. In fact, another asset whose transfer is not blocked should be able to liquidate.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L250-L255

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L367-L385

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/Account.sol#L163-L174

## Tool used

Manual Review

## Recommendation

Liquidation and borrowing should be done per asset instead of a global pooling of all assets. This will limit the damage to just the asset whose transfer is blocked.
