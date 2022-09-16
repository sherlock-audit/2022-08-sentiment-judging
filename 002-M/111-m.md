0xNazgul
# [NAZ-M4] Chainlink's `latestRoundData` Might Return Stale Results

## Summary
There is use of Chainlink's `latestRoundData` with missing checks the would ensure the prevention of Stale Results.

## Vulnerability Detail
Across these contracts, we are using Chainlink's `latestRoundData` API, but there is only a check if the return value is `< 0` and could then be 0. This could lead to stale prices according to the Chainlink documentation:

* [Historical Price data](https://docs.chain.link/docs/historical-price-data/#historical-rounds)
* [Checking Your returned answers](https://docs.chain.link/docs/faq/#how-can-i-check-if-the-answer-to-a-round-is-being-carried-over-from-a-previous-round)

## Impact
The result of `latestRoundData` API will be used across various functions, therefore, a stale price from Chainlink can lead to loss of funds to end-users.

## Code Snippet
[`ArbiChainlinkOracle.sol#L51`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/oracle/src/chainlink/ArbiChainlinkOracle.sol#L51), [`ChainlinkOracle.sol#L51`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/oracle/src/chainlink/ChainlinkOracle.sol#L51)

## Tool used
Manual Review

## Recommendation
Consider adding the missing checks for stale data.

For example:
```js
(uint80 roundID ,answer,, uint256 timestamp, uint80 answeredInRound) = AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData();

require(answer > 0, "Chainlink price <= 0"); 
require(answeredInRound >= roundID, "Stale price");
require(timestamp != 0, "Round not complete");
```