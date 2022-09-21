Olivierdem
# Medium Risk: Oracle data feed is insufficiently validated

## Summary

Oracle data feed is insufficiently validated in "oracle/src/chainlink/ChainlinkOracle.sol"

## Vulnerability Detail

Oracle data feed is insufficiently validated. There is no check for stale price and round completeness.
Price can be stale and can lead to wrong getEthPrice return value.

- Validate data feed. Chainlink's call to 'ethUsdPriceFeed.latestRoundData();' returns 5 values.
  It is therefore imprudent to ignore 4 of them.

```
function getEthPrice() internal view returns (uint) {
      (, int answer,,,) =
          ethUsdPriceFeed.latestRoundData();
      if (answer < 0)
          revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

      return uint(answer);
  }
```

## Impact

Medium to High since it impact the price

## Code Snippet

## Tool used

Manual Review

## Recommendation

The oracle returns 5 values, so it's reasonable to use more than just one to be sure the price isn't stale:

```
    function getEthPrice() internal view returns (uint) {
         (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) =
             ethUsdPriceFeed.latestRoundData();

         if (answer < 0)
             revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));
         if (answeredInRound >= roundID)
             revert Errors.StaleEthPrice("ChainLink: Stale ETH price");
         if (timestamp > 0)
             revert Errors.RoundNotCompleted("ChainLink: ETH round not complete"));

         return uint(answer);
     }
```
