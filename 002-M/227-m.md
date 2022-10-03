cccz
# Chainlink's latestRoundData might return stale or incorrect results

## Summary
Chainlink's latestRoundData might return stale or incorrect results
## Vulnerability Detail
On ChainlinkOracle/ArbiChainlinkOracle, we are using latestRoundData, but there is no check if the return value indicates stale data.
```
    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();
```
This could lead to stale prices according to the Chainlink documentation:

https://docs.chain.link/docs/historical-price-data/#historical-rounds
https://docs.chain.link/docs/faq/#how-can-i-check-if-the-answer-to-a-round-is-being-carried-over-from-a-previous-round
## Impact
Stale data from the oracle may make the calculations in the RiskEngine contract incorrect.
## Code Snippet
```
    function getPrice(address token) external view override returns (uint) {
        if (!isSequencerActive()) revert Errors.L2SequencerUnavailable();

        (, int answer,,,) =
            feed[token].latestRoundData();
...
    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();
...

    function getEthPrice() internal view returns (uint) {
        (, int answer,,,) =
            ethUsdPriceFeed.latestRoundData();
...
```
## Tool used

Manual Review

## Recommendation
Consider adding the following check to the results of latestRoundData()
```
    (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) = feed.latestRoundData();
    require(answeredInRound >= roundID, "Stale price");
    require(timestamp != 0,"Round not complete");
    require(answer > 0,"Chainlink answer reporting 0");
```