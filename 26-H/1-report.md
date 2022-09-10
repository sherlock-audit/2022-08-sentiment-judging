cergyk
# UniV2LPOracle: Decimals of pair tokens ignored

## Summary
`UniV2LPOracle` does not take into account how many decimals the reserve tokens have, leading to wrong values.

## Vulnerability Detail
`getReserves()` returns the tokens in their decimals. Therefore, we have in the calculation (see the comments for the calculations):
```
        return FixedPointMathLib.sqrt(
            r0 //d1 decimals
            .mulWadDown(r1) //d2 decimals
            .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token0())) //18 decimals
            .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token1())) //18 decimals
        ) // (d1 + d2 + 36)/2 decimals
        .mulDivDown(2e27, IUniswapV2Pair(pair).totalSupply()); // (d1 + d2 + 36)/2 + 27 - 18 decimals
```
Meaning that the returned value will not have 18 decimals.

## Impact
The price that is used for those LP tokens will be way too high, which can be exploited to create debt positions with a health ratio below 1.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/uniswap/UniV2LPOracle.sol#L49

## Tool used

Manual Review

## Recommendation
Normalize (taking into account the decimals of the reserves) to have a result with 18 decimals.