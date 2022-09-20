__141345__
# UniSwapV3 oracle `twapPeriod` update need time lock

## Summary

The UniSwapV3 oracle `twapPeriod` update can take effects immediately, the number used by inner accounting will change right away, some user account close to the liquidation threshold might be underwater following the update. Once liquidated, these users will loss fund.

A time lock for these sensitive parameter change can help.

## Impact

Some user might be liquidated immediately after the `twapPeriod` update and lose fund. 


## Code Snippet

The `twapPeriod` update can take effects immediately.
```solidity
// oracle\src\uniswap\UniV3TWAPOracle.sol
    function updateTwapPeriod(uint32 _twapPeriod) external adminOnly {
        twapPeriod = _twapPeriod;
    }
```

As a result, the oracle price feed based on `twapPeriod` will also change all of a sudden.
```solidity
    function getPrice(address token) public view returns (uint256) {
        // ...
        (int24 arithmeticMeanTick, ) = OracleLibrary.consult(
            pool,
            twapPeriod
        );
        // ...
    }
```

## Tool used

Manual Review

## Recommendation

Add a time lock for `twapPeriod` update, allow all users to have enough time to react.

