0xf15ers
#  Oracle `latestRoundData` might return stale or incorrect results

## Summary
There is no checks performed for stale data. 
## Vulnerability Detail

## Impact
This could lead to using historical stale result. 

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-xremora/blob/8801afd6e03e8211e2804caa39c0255eba37a8f4/oracle/src/chainlink/ChainlinkOracle.sol#L49-L51 


```solidity
function getEthPrice() internal view returns (uint) {
    (, int answer,,,) =
        ethUsdPriceFeed.latestRoundData();
.....................
```


## Tool used

Manual Review

## Recommendation
- add validation for 
```solidity
function getEthPrice() internal view returns (uint) {
(uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) = ethUsdPriceFeed.latestRoundData();
require(answeredInRound >= roundID, "Stale price");

.....................
```