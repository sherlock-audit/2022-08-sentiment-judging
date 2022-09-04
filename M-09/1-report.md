defsec
# Should check return data from chainlink aggregators

## Summary

The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to protocol.

## Severity
Medium

## Vulnerability Detail

The getPrice function in the contract ChainlinkOracle.sol fetches the asset price from a Chainlink aggregator using the latestRoundData function. However, there are no checks on roundID nor timeStamp, resulting in stale prices. The oracle wrapper calls out to a chainlink oracle receiving the latestRoundData(). It then checks freshness by verifying that the answer is indeed for the last known round. The returned updatedAt timestamp is not checked.

## Impact

Stale prices could put funds at risk. According to Chainlink's documentation, This function does not error if no answer has been reached but returns 0, causing an incorrect price fed to the PriceOracle. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to protocol

## Code Snippet

[ChainlinkOracle.sol#L67](https://github.com/sherlock-audit/2022-08-sentiment-defsec/blob/main/oracle/src/chainlink/ChainlinkOracle.sol#L67)

## Tool used
Manual Review

## Recommendation

Consider to add checks on the return data with proper revert messages if the price is stale or the round is incomplete, for example:

```solidity
(uint80 roundID, int256 price, , uint256 timeStamp, uint80 answeredInRound) = ETH_CHAINLINK.latestRoundData();
require(price > 0, "Chainlink price <= 0"); 
require(answeredInRound >= roundID, "...");
require(timeStamp != 0, "...");
```

## Team  
-

## Sherlock  
- 
