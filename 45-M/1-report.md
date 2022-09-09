minera
# Oracle Consideration: Curve price oracle can return invalid price data.

## Summary

the curve oracle in 

``` 
 Stable2CurveOracle.sol
```

the oracle data can be rounded to 0 or return invalid data.

## Vulnerability Detail

In Stable2CurveOracle.sol

the getPrice is implemented as shown below.

```
    /// @inheritdoc IOracle
    function getPrice(address token) external view returns (uint) {
        uint price0 = oracleFacade.getPrice(ICurvePool(token).coins(0));
        uint price1 = oracleFacade.getPrice(ICurvePool(token).coins(1));
        return ((price0 < price1) ? price0 : price1).mulWadDown(
            ICurvePool(token).get_virtual_price()
        );
    }
```

we get price0 and price1 and use whatever the small number is and divde by virtual price.

When price0 and price1 has a huge price difference (when the curve pool in very imbalanced), the above approach can report undervalued oracle data.

Or if both price0 and price1 is smaller than the virtual price,

because of the division logic

```
 ((price0 < price1) ? price0 : price1).mulWadDown(ICurvePool(token).get_virtual_price())
```

the oracle data would be rounded to 0.

## Impact

If the oracle data is invalid because of imbalanced token pool or very small price0 / price1, then the function _valueIn can get the invalid or lagging oracle and determine the wrong number of debt or callateral worth. User may not able to deposit or repay to manage their position or malicious liquidators can liquidated user's account balance.

```
    function _valueInWei(address token, uint amt)
        internal
        view
        returns (uint)
    {
        return oracle.getPrice(token)
        .mulDivDown(
            amt,
            10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
        );
    }
```

## Code Snippet

## Tool used

Manual Review

Yes

## Recommendation

To avoid price manipulation, using the virtual price would be sufficient,

https://news.curve.fi/chainlink-oracles-and-curve-pools/

```
    function getPrice(address token) external view returns (uint) {
        return  ICurvePool(token).get_virtual_price();
    }
```
