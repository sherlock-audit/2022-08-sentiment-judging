Lambda
# Stable2CurveOracle: Missing normalization

## Summary
The price that is returned by `getPrice()` of `Stable2CurveOracle` is not normalized to 10^18, leading to wrong values for those tokens.

## Vulnerability Detail
`oracleFacade.getPrice(ICurvePool(token).coins(0))` returns a price with 18 decimals. However, `ICurvePool(token).get_virtual_price()` also returns a price that is normalized to 18 decimals. This is for instance visible in the source code of the stETH / ETH pool: `@return LP token virtual price normalized to 1e18` (https://etherscan.io/address/0xDC24316b9AE028F1497c275EB9192a3Ea0f67022#code)

## Impact
The returned prices for the LP tokens will have 36 decimals. For instance, the stETH / ETH `get_virtual_price()` at the time of writing is 1049449001674373194. Assuming stETH is 0.95 ETH (9.5 * 1e17), the returned value will be:
1049449001674373194 * 9.5 * 1e17 ~= 1e36

This is wrong and will cause those LP tokens to be significantly overvalued, which can be exploited to take out debt positions where the health ratio (if properly calculated) would be way below 1.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/curve/Stable2CurveOracle.sol#L47

## Tool used

Manual Review

## Recommendation
```
return ((price0 < price1) ? price0 : price1).mulWadDown(
            ICurvePool(token).get_virtual_price()
           ).divWadDown(1e18);
```
This will correctly return ~1e18 in the above example, which is more or less the current price (1 ETH) of this LP token.
