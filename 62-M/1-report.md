minera
# Missing verification in `getEthPrice` and `getPrice` could lead to stale/incorrect price

## Summary

On ChainlinkOracle.sol, there are missing checks to verify if price is stale/incorrect.

## Vulnerability Detail

In `latestRoundData` function there is a need to check `answeredInRound` , `roundID` and `timestamp` to check if the price could be stale.

## Impact

This could lead to incorrect/stale prices that may cause invalid calculation and a possible wrong liquidation, losing user's funds.

## Code Snippet

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
  (, int answer,,,) = ethUsdPriceFeed.latestRoundData();

  if (answer < 0)
    revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

  return uint(answer);
}
```

## Tool used

Manual Review

## Recommendation

The recommendation is to follow the code:

```diff
function getEthPrice() internal view returns (uint) {
-  (, int answer,,,) = ethUsdPriceFeed.latestRoundData();
+  (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) =
     ethUsdPriceFeed.latestRoundData();

  if (answer < 0)
    revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

+	if (answeredInRound < roundID) 
+	  revert Errors.StalePrice(address(0), address(ethUsdPriceFeed));

+ if(timestamp == 0) 
+    revert Errors.RoundIncomplete(address(0), address(ethUsdPriceFeed));

  return uint(answer);
}
```