xiaoming90
# Chainlink's LatestRoundData Might Return Stale Results

## Summary

The oracle might return a stale price due to a lack of validation.

## Vulnerability Detail

In a best-case scenario, rounds update chronologically. However, a round can time out if it doesn't reach a consensus. Technically, that is a timed-out round that carries over the answer from the previous round.

If the contract called `latestRoundData()` function and the price returned from the Oracle is carried over from the previous round, the contract might be using a stale price for calculation, which might cause some issues.

The following is an extract from [Chainlink ](https://docs.chain.link/docs/historical-price-data/)site

> The `updatedAt` data feed property is the timestamp of an answered round and `answeredInRound` is the round it was updated in. You can check `answeredInRound` against the current `roundId`. If `answeredInRound` is less than `roundId`, the answer is being carried over. If `answeredInRound` is equal to `roundId`, then the answer is fresh.

It was observed that the `getPrice` and `getEthPrice` functions use the `latestRoundData` function, but the functions did not validate if the returned values indicate stale data.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/oracle/src/chainlink/ChainlinkOracle.sol#L49

```solidity
/// @inheritdoc IOracle
/// @dev feed[token].latestRoundData should return price scaled by 8 decimals
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

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/oracle/src/chainlink/ChainlinkOracle.sol#L65

```solidity
function getEthPrice() internal view returns (uint) {
    (, int answer,,,) =
        ethUsdPriceFeed.latestRoundData();

    if (answer < 0)
        revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

    return uint(answer);
}
```

## Impacts

Stale prices that do not reflect the current market price. This will cause the user's free collateral to be valued wrongly leading to wrong liquidation or borrowing.

## Recommendation

Consider implementing additional checks to ensure that the returned values are not stale:

```diff
function getPrice(address token) external view virtual returns (uint) {
-   (, int answer,,,) =
-       feed[token].latestRoundData();
+   (uint roundID, int answer,, uint answeredInRound) =
+    	feed[token].latestRoundData();

+	if (answeredInRound < roundID)
+		revert Errors.StalePrice
    if (answer < 0)
        revert Errors.NegativePrice(token, address(feed[token]));


    return (
        (uint(answer)*1e18)/getEthPrice()
    );
}

function getEthPrice() internal view returns (uint) {
-   (, int answer,,,) =
-       ethUsdPriceFeed.latestRoundData();
+   (uint roundID, int answer,, uint answeredInRound) =
+    	ethUsdPriceFeed.latestRoundData();

+	if (answeredInRound < roundID)
+		revert Errors.StalePrice
    if (answer < 0)
        revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

    return uint(answer);
}
```