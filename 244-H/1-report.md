Bahurum
# Missing validation of `latestRoundData` return data

## Summary
`AggregatorV3.latestRoundData` could return stale or incorrect prices. This can lead to incorrect margin calculation allowing to steal borrowed funds.

## Vulnerability Detail
In [ArbiChainlinkOracle.sol#L50-L51](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ArbiChainlinkOracle.sol#L50-L51) and [ChainlinkOracle.sol#L50-L51](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L50-L51) price freshness is not checked. In case of network congestion or high volatility of an asset the feed's `answer` can be outdated respect to the current market price. In such cases the price used for margin calculation is off and allows theft of loan if it is off by more than approx. 20%. A clear case is when an asset plummets and the minimum value circuit breaker triggers. See what happened during the LUNA crash when LUNA/USD feed reported a price 1000% higher than the market and some lending protocols that didn't check for staleness got drained.

## Impact
Loan can be stolen in case of high volatility.

## Code Snippet
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L48-L59

```solidity

    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```

## Tool used

Manual Review

## Recommendation
These are the return values of `latestRoundData`
``` solidity
function latestRoundData() external view
    returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    )
```
on should check that `roundId == answeredInRound` and that `updatedAt` is not older than a given delay. 

Do not revert in case these are not true since the price feed is also used to determine liquidatability and liquidations would revert during high volatility leading to losses for the protocol.
Define a variable `isStale` and return it with the price. In all checks of account margin except for liquidations revert if `isStale`.

```diff

-   function getPrice(address token) external view virtual returns (uint) {
+   function getPrice(address token) external view virtual returns (uint, bool) {
-       (, int answer,,,) =
+       (uint80 roundId,
+       int256 answer,
+       uint256 startedAt,
+       uint256 updatedAt,
+       uint80 answeredInRound) =
            feed[token].latestRoundData();

+       bool isStale = 
+           (roundId != answeredInRound)||(block.timestamp > updatedAt + DELAY)
        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
-           (uint(answer)*1e18)/getEthPrice()            
+           ((uint(answer)*1e18)/getEthPrice(), isStale)
        );
    }
```