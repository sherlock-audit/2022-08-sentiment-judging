GalloDaSballo
# M-02 CTokenOracle may use stale interest rate

## Summary

[`CTokenOracle.getCErc20Price`](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/compound/CTokenOracle.sol#L66) is using `exchangeRateStored` to maintain `view` visibility.

This will cause a difference between the rate that the Oracle Will report, and the actual rate that the token will have during a `redeem` or when levering up.

This is because `exchangeRateStored` is updated after each `accrue` which requires a non-view call.

The risk with this is that the CTokenOracle will report the incorrect CToken ExchangeRate as the actual rate may be considerably different, especially when considering a system that needs to handle debt and account health.

## Impact

The oracle will report the incorrect exchangeRate, the reported exchangeRate and the actual rate will be different.

Based on available system liquidity this could be used by an attacker to arbitrage the system and extract more value than expected.

## Code Snippet

This problem has been solved by t11s, you can see `viewExchangeRate` for a gas efficient way to retrieve the most up to date exchangeRate

See: https://github.com/transmissions11/libcompound/blob/20ab8e325440749d56cde122bd21bd55f5f6e9d8/src/LibCompound.sol#L17

## Tool used

Manual Review

## Recommendation

Use:
https://github.com/transmissions11/libcompound/blob/20ab8e325440749d56cde122bd21bd55f5f6e9d8/src/LibCompound.sol#L17


To maintain the code as view, but remove the risk of arbitrage