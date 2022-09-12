csanuragjain
# Improper Validation Of latestRoundData Function

## Summary
Additional checks on latestRoundData is missing

## Vulnerability Detail
1. Necessary checks are missing while using [latestRoundData](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L51) function which means the returned price might be incorrect

## Impact
Price returned by getPrice function might be incorrect

## Code Snippet
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ArbiChainlinkOracle.sol#L47

## Tool used
Manual Review

## Recommendation
Change the function implementation as below:

```
function getPrice(address token) external view virtual returns (uint) {
        (uint80 roundID, int answer,,uint256 updatedAt,uint80 answeredInRound) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));
require(
          answeredInRound >= roundID,
          "Price Stale"
      );
      require(answer > 0, "Malfunction");
      require(updatedAt != 0, "Incomplete round");
        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```