Chom
# Too many assets can't be liquidated if an oracle is down

## Summary
Too many assets can't be liquidated if an oracle is down

## Vulnerability Detail

1. Oracle down (oracle.getPrice reverted).
2. Can't calculate risk ratio as isAccountHealthy calls _getBalance and _getBorrows which use reverted oracle.getPrice.
3. Can't liquidate since risk ratio can't be calculated.

This happened during Luna / UST crisis.

## Impact
Too many assets can't be liquidated if an oracle is down. If that user uses an asset that oracle is down as collateral and uses another asset that price is crashing but oracle is not down as collateral too, that user can't be liquidated. This will cause a very serious bad debt. In fact, another asset whose price is crashing but oracle is not down should be able to liquidate.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L250-L255

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L367-L385

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/RiskEngine.sol#L121-L125

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/RiskEngine.sol#L190-L197

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/RiskEngine.sol#L150-L161

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/RiskEngine.sol#L163-L176

## Tool used

Manual Review

## Recommendation
Liquidation and borrowing should be done per asset instead of a global pooling of all assets. This will limit the damage to just the asset which oracle is down.