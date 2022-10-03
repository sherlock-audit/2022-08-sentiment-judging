Chom
# OracleLibrary consult may overflow. Once it is overflow, the oracle will revert.

## Summary
OracleLibrary consult may overflow. Once it is overflow, the oracle will revert.

## Vulnerability Detail
OracleLibrary is copied from https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol which only works on solidity <0.8.0 but OracleLibrary uses solidity 0.8.15

```solidity
        (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s) =
            IUniswapV3Pool(pool).observe(secondsAgos);

        int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
        uint160 secondsPerLiquidityCumulativesDelta =
            secondsPerLiquidityCumulativeX128s[1] - secondsPerLiquidityCumulativeX128s[0];
```

Cumulative is usually overflow especially if it uses int56. Read Notes on overflow section of https://docs.uniswap.org/protocol/V2/guides/smart-contract-integration/building-an-oracle (This applies to all cumulative-based oracle not just Uniswap V2).tickCumulatives[1] - tickCumulatives[0] will be underflow and revert.

Once it overflows, tickCumulatives[1] may less than tickCumulatives[0] (tickCumulatives[1] ~ -type(int56).max, tickCumulatives[0] ~ type(int56).max). In this case, the consult will revert.

## Impact
OracleLibrary consult may overflow. Once it is overflow, the oracle will revert. These functions but are not limited to may be broken.

- isAccountHealthy may be broken as it calls _getBalance and _getBorrows which use reverted oracle.getPrice.
- liquidate may be broken as a result of isAccountHealthy broken. This will cause bad debt!

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/oracle/src/uniswap/library/OracleLibrary.sol#L22-L41

## Tool used

Manual Review

## Recommendation

Add unchecked block around consult functions to mimic the behavior of solidity <0.8.0

```solidity
    function consult(address pool, uint32 secondsAgo)
        internal
        view
        returns (int24 arithmeticMeanTick, uint128 harmonicMeanLiquidity)
    {
      unchecked {
        require(secondsAgo != 0, 'BP');

        uint32[] memory secondsAgos = new uint32[](2);
        secondsAgos[0] = secondsAgo;
        secondsAgos[1] = 0;

        (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s) =
            IUniswapV3Pool(pool).observe(secondsAgos);

        int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
        uint160 secondsPerLiquidityCumulativesDelta =
            secondsPerLiquidityCumulativeX128s[1] - secondsPerLiquidityCumulativeX128s[0];

        arithmeticMeanTick = int24(tickCumulativesDelta / int32(secondsAgo));
        // Always round to negative infinity
        if (tickCumulativesDelta < 0 && (tickCumulativesDelta % int32(secondsAgo) != 0)) arithmeticMeanTick--;

        // We are multiplying here instead of shifting to ensure that harmonicMeanLiquidity doesn't overflow uint128
        uint192 secondsAgoX160 = uint192(secondsAgo) * type(uint160).max;
        harmonicMeanLiquidity = uint128(secondsAgoX160 / (uint192(secondsPerLiquidityCumulativesDelta) << 32));
      }
    }
```