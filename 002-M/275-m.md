WATCHPUG
# Lack of price freshness check in `ChainlinkOracle.sol#getPrice()` allows a stale price to be used

## Summary

`ChainlinkOracle` should use the `updatedAt` value from the latestRoundData() function to make sure that the latest answer is recent enough to be used.

## Vulnerability Detail

In the current implementation of `ChainlinkOracle.sol#getPrice()`, there is no freshness check. This could lead to stale prices being used.

If the market price of the token drops very quickly ("flash crashes"), and Chainlink's feed does not get updated in time, the smart contract will continue to believe the token is worth more than the market value.

Chainlink also advise developers to check for the `updatedAt` before using the price:

> Your application should track the latestTimestamp variable or use the updatedAt value from the latestRoundData() function to make sure that the latest answer is recent enough for your application to use it. If your application detects that the reported answer is not updated within the heartbeat or within time limits that you determine are acceptable for your application, pause operation or switch to an alternate operation mode while identifying the cause of the delay.

And they have this heartbeat concept:

> Chainlink Price Feeds do not provide streaming data. Rather, the aggregator updates its latestAnswer when the value deviates beyond a specified threshold or when the heartbeat idle time has passed. You can find the heartbeat and deviation values for each data feed at data.chain.link or in the Contract Addresses lists.

The `Heartbeat` on Arbitrum is usually `1h`.

Source: https://docs.chain.link/docs/arbitrum-price-feeds/

## Impact

A stale price can cause the malfunction of multiple features across the protocol:

1. `_valueInWei()` is using the price to calculate the value of the loan and collateral; A stale price will make the price calculation inaccurate so that certain accounts may not be liquidated when they should be, or be liquidated when they should not be.

2. `ChainlinkOracle.sol#getPrice()` is used to calculate the value of various LPTokens (Aave, Balancer, Compound, Curve, and Uniswap). If the price is not accurate, it will lead to a deviation in the LPToken price and affect the calculation of asset prices.

3. Stale asset prices can lead to bad debts to the protocol as the collateral assets can be overvalued, and the collateral value can not cover the loans.

## Code Snippet

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L59

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
```

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L65-L73

```solidity
    function getEthPrice() internal view returns (uint) {
        (, int answer,,,) =
            ethUsdPriceFeed.latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

        return uint(answer);
    }
```

## Tool used

Manual Review

## Recommendation

Consider adding the missing freshness check for stale price:

```solidity
    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();
        uint validPeriod = feedValidPeriod[token];

        require(block.timestamp - updatedAt < validPeriod, "freshness check failed.")

        if (answer <= 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```

The `validPeriod` can be based on the `Heartbeat` of the feed.