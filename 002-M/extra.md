Dravee
# Chainlink's `latestRoundData` might return stale or incorrect results


## Affected code

https://github.com/sherlock-audit/2022-08-sentiment-JustDravee/blob/34c01a8606a02a6d9ee27b39a46572c06ea33a15/oracle/src/chainlink/ArbiChainlinkOracle.sol#L50-L54

https://github.com/sherlock-audit/2022-08-sentiment-JustDravee/blob/34c01a8606a02a6d9ee27b39a46572c06ea33a15/oracle/src/chainlink/ArbiChainlinkOracle.sol#L66-L68

https://github.com/sherlock-audit/2022-08-sentiment-JustDravee/blob/34c01a8606a02a6d9ee27b39a46572c06ea33a15/oracle/src/chainlink/ChainlinkOracle.sol#L50-L54

https://github.com/sherlock-audit/2022-08-sentiment-JustDravee/blob/34c01a8606a02a6d9ee27b39a46572c06ea33a15/oracle/src/chainlink/ChainlinkOracle.sol#L66-L70

## Vulnerability Detail
`latestRoundData()` is used to fetch the asset price from a Chainlink aggregator, but it's missing additional validations to ensure that the round is complete. If there is a problem with Chainlink starting a new round and finding consensus on the new value for the oracle (e.g. Chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the Chainlink system) consumers of this contract may continue using outdated stale data / stale prices.

These properties are important to check:

- The `updatedAt` data feed property is the timestamp of an answered round and answeredInRound is the round it was updated in. A timestamp with zero value means the round is not complete and should not be used.
- If the `answeredInRound` data feed property is less than the `roundID` data feed property, the `answer` is being carried over. If `answeredInRound` is equal to `roundID`, then the `answer` is fresh.

Consider adding their missing checks for stale data.

As an example:

- [ArbiChainlinkOracle.sol#getPrice()](https://github.com/sherlock-audit/2022-08-sentiment-JustDravee/blob/34c01a8606a02a6d9ee27b39a46572c06ea33a15/oracle/src/chainlink/ArbiChainlinkOracle.sol#L50-L54)
```diff
- 50:         (, int answer,,,) =
+ 50:         (uint80 roundID, int answer,, uint256 updatedAt, uint80 answeredInRound) =
51:             feed[token].latestRoundData();
52:
53:         if (answer < 0)
54:             revert Errors.NegativePrice(token, address(feed[token]));
+ 55:         if (updatedAt == 0 || answeredInRound < roundID)
+ 56:             revert Errors.StalePriceData();
```