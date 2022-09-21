Chom
# Missing UniswapV2 TWAP oracle implementation

## Summary
Missing UniswapV2 TWAP oracle implementation

## Vulnerability Detail
```solidity
    /// @dev Adapted from https://blog.alphaventuredao.io/fair-lp-token-pricing
    /// @inheritdoc IOracle
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
```

UniV2LPOracle getPrice requires underlying tokens in pairs to have a separate oracle such as Chainlink. But many tokens in UniswapV2 never have a price in any separate oracle. These tokens require UniswapV2 TWAP oracle implementation.

## Impact
Missing UniswapV2 TWAP oracle implementation. Governance tokens that aren't supported by any Oracle such as Chainlink won't have an oracle available. Without TWAP oracle, these tokens easily get attacked by a flash loan.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/oracle/src/uniswap/UniV2LPOracle.sol#L39-L50

## Tool used

Manual Review

## Recommendation

Please implement TWAP oracle for UniswapV2 according to this example. Remember to put unchecked blocks around these functions too since this code is written before solidity 0.8.0.

https://github.com/Uniswap/v2-periphery/blob/master/contracts/examples/ExampleOracleSimple.sol

https://docs.uniswap.org/protocol/V2/guides/smart-contract-integration/building-an-oracle
