ellahi
# Oracle data feed is insufficiently validated.

## Summary
Chainlink oracle data feed is insufficiently validated. Price can be stale and can lead to wrong return value. This would result in incorrect accounting done in `RiskEngine.sol`.
## Vulnerability Detail
Oracle data feed is insufficiently validated. There is no check for stale price and round completeness.
## Impact
Medium
## Code Snippet
[`ChainlinkOracle.sol::getPrice()`](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L59), [`ChainlinkOracle.sol::getEthPrice()`](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L65-L73).
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
[`ArbiChainlinkOracle.sol::getPrice()`](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ArbiChainlinkOracle.sol#L47-L59).
```solidity
function getPrice(address token) external view override returns (uint) {
    if (!isSequencerActive()) revert Errors.L2SequencerUnavailable();

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
Validate data feed. Consider implementing a similar solution like the following code snippet:
```solidity
function getPrice(address token) external view virtual returns (uint) {
    (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) = feed[token].latestRoundData();
    if (answer < 0) revert Errors.NegativePrice(token, address(feed[token]));
    if (answeredInRound < roundID) revert ... // Stale price
    if (timestamp == 0) revert ... // Token round not complete
    ...
```