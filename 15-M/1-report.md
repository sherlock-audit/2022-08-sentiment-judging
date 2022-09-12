oyc_109
# Chainlink oracle aggregator data is insufficiently validated

## Summary

https://github.com/sherlock-audit/2022-08-sentiment-andyfeili/blob/96338b720493bc6dcbfa8ed24b75af53adc7900d/oracle/src/chainlink/ChainlinkOracle.sol#L49-L73

## Vulnerability Detail

The function getPrice() and getEthPrice() fetches the latestRoundData() from a Chainlink oracle feed. However, neither round completeness or the quoted timestamp are checked to ensure that the reported price is not stale.

## Recommendation

Add additional validation to check if the price is stale and round is complete

eg.
```solidity
 (uint80 roundID, int256 answer, , uint256 updatedAt, uint80 answeredInRound) = feed.latestRoundData();
require(answeredInRound >= roundID, "ChainLink: Stale price");
require(updatedAt != 0, "ChainLink: Round not complete");
```