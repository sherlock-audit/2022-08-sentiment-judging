0x52
# UniV2LPOracle.sol incorrectly values LP when either token in a pair does not have 18 decimals

## Summary

UniV2LPOracle.sol incorrectly values LP when either token doesn't have 18 decimals

## Vulnerability Detail

    function getPrice(address pair) external view returns (uint) {
        (uint r0, uint r1,) = IUniswapV2Pair(pair).getReserves();


        // 2 * sqrt(r0 * r1 * p0 * p1) / totalSupply
        return FixedPointMathLib.sqrt(
            r0
            .mulWadDown(r1)
            .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token0()))
            .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token1()))
        )
        .mulDivDown(2e27, IUniswapV2Pair(pair).totalSupply());
    }

UniV2LPOracle.sol#getPrice is hard coded to only work when the decimals of both tokens in a pair are 18. This limitation is a result of using a constant 2e27 in L49 and oracle prices that are fixed to 18 decimals. Any variation from this causes LP to be valued incorrectly; especially important as USDC is commonly in pairs and only has 6 decimals. 

## Impact

Incorrect valuation of LP, potentially leading to unfair liquidation

## Code Snippet

[UniV2LPOracle.sol#L39-L50](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/uniswap/UniV2LPOracle.sol#L39-L50)

## Tool used

Manual Review

## Recommendation

Standardize all token balances to 18 decimals before doing any calculations:

    r0 = r0 * 10 ** (18 - IUniswapV2Pair(pair).token0().decimals());
    r1 = r1 * 10 ** (18 - IUniswapV2Pair(pair).token1().decimals());