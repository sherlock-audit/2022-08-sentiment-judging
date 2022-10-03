devtooligan
# Oracle data feed is insufficiently validated - MEDIUM

## Summary

Price can be stale and can lead to inaccurate prices affecting borrows, liquidations, and withdraws.

## Vulnerability Detail
Chainlink price feeds offer timestamp data when querying price but this is not being validated by the oracle.  

## Impact
Medium - This could affect `borrow()`, `withdraw()` and `liquidate()` depending on the severity of the price difference.

## Code Snippet
<details>
  <summary> ChainlinkOracle.getPrice()
</summary>

```
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
</details>

## Tool used

Manual Review

## Recommendation

it is important to ensure the price feed data was updated recently.  As such, recommend validating the `roundID`, `answeredInRound` and `timestamp` as follows:

```diff
diff --git a/oracle/src/chainlink/ChainlinkOracle.sol b/oracle/src/chainlink/ChainlinkOracle.sol
index d83be8d..7c37d7d 100644
--- a/oracle/src/chainlink/ChainlinkOracle.sol
+++ b/oracle/src/chainlink/ChainlinkOracle.sol
@@ -47,9 +47,12 @@ contract ChainlinkOracle is Ownable, IOracle {
     /// @inheritdoc IOracle
     /// @dev feed[token].latestRoundData should return price scaled by 8 decimals
     function getPrice(address token) external view virtual returns (uint) {
-        (, int answer,,,) =
+        (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) =
             feed[token].latestRoundData();

+        require(answeredInRound >= roundID, "ChainLink: Stale price");
+        require(timestamp > 0, "ChainLink: Round not complete");
+
         if (answer < 0)
             revert Errors.NegativePrice(token, address(feed[token]));

@@ -63,9 +66,12 @@ contract ChainlinkOracle is Ownable, IOracle {
     /* -------------------------------------------------------------------------- */

     function getEthPrice() internal view returns (uint) {
-        (, int answer,,,) =
+        (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) =
             ethUsdPriceFeed.latestRoundData();

+        require(answeredInRound >= roundID, "ChainLink: Stale price");
+        require(timestamp > 0, "ChainLink: Round not complete");
+
         if (answer < 0)
             revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

```
