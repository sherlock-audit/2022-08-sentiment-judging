0x52
# CTokenOracle.sol#getCErc20Price contains critical math error

## Summary

CTokenOracle.sol#getCErc20Price contains a math error that immensely overvalues CTokens

## Vulnerability Detail

[CTokenOracle.sol#L66-L76](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/compound/CTokenOracle.sol#L66-L76)

    function getCErc20Price(ICToken cToken, address underlying) internal view returns (uint) {
        /*
            cToken Exchange rates are scaled by 10^(18 - 8 + underlying token decimals) so to scale
            the exchange rate to 18 decimals we must multiply it by 1e8 and then divide it by the
            number of decimals in the underlying token. Finally to find the price of the cToken we
            must multiply this value with the current price of the underlying token
        */
        return cToken.exchangeRateStored()
        .mulDivDown(1e8 , IERC20(underlying).decimals())
        .mulWadDown(oracle.getPrice(underlying));
    }

In L74, IERC20(underlying).decimals() is not raised to the power of 10. The results in the price of the LP being overvalued by many order of magnitudes. A user could deposit one CToken and drain the reserves of every liquidity pool.

## Impact

All lenders could be drained of all their funds due to excessive over valuation of CTokens cause by this error

## Code Snippet

[CTokenOracle.sol#L66-L76](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/compound/CTokenOracle.sol#L66-L76)

## Tool used

Manual Review

## Recommendation

Fix the math error by changing L74:

    return cToken.exchangeRateStored()
    .mulDivDown(1e8 , 10 ** IERC20(underlying).decimals())
    .mulWadDown(oracle.getPrice(underlying));
       