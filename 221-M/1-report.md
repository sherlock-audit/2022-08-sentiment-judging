__141345__
# Chainlink `latestRoundData()` might return stale or incorrect results

## Summary

Chainlink `latestRoundData()` might return stale or incorrect results.

This could lead to stale prices and wrong price return value, or outdated price. The functions rely on accurate price feed might not work as expected, sometimes can lead to fund loss. 


## Vulnerability Detail

The oracle wrapper `getPrice(token)` and `getEthPrice()` call out to a chainlink oracle with `feed.latestRoundData()` to get the price of some token or native ETH. But there is no check if the return value indicates stale data and round completeness.

According to Chainlink's documentation, this function does not error if no answer has been reached but returns 0 or outdated round data. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to the index price calculations. Oracle reliance has historically resulted in crippled on-chain systems, and complications that lead to these outcomes can arise from things as simple as network congestion.

## Reference
Chainlink documentation:
https://docs.chain.link/docs/historical-price-data/#historical-rounds

## Impact

If there is a problem with chainlink starting a new round and finding consensus on the new value for the oracle (e.g. chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) consumers of this contract may continue using outdated stale data (if oracles are unable to submit no new round is started).

This could lead to stale prices and wrong price return value, or outdated price.

As a result, the functions rely on accurate price feed might not work as expected, sometimes can lead to fund loss. The impacts vary and depends on the specific situation like the following:
- incorrect liquidation
    - some users could be liquidated when they should not
    - no liquidation is performed when there should be
- wrong price feed 
    - causing inappropriate loan being taken, beyond the current collateral factor
    - too low price feed affect normal bor


## Code Snippet

ChainlinkOracle.sol
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

    function getEthPrice() internal view returns (uint) {
        (, int answer,,,) =
            ethUsdPriceFeed.latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

        return uint(answer);
    }
```


## Tool used

Manual Review


## Recommendation

Validate data feed:
```solidity
    (uint80 roundID, int256 answer, , uint256 updatedAt, uint80 answeredInRound) = feed[token].latestRoundData();
    
    if (answer < 0)
        revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));
    if (block.timestamp - updatedAt < SECONDS_PER_HOUR) revert Errors.RoundIncompleted();
    if (answeredInRound < roundID) revert Errors.Stale price();
```