pashov
# Incomplete price validation for Chainlink’s `latestRoundData` in `ChainlinkOracle.sol` & `ArbiChainlinkOracle.sol` can lead to overleveraged borrowing

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ChainlinkOracle.sol#L49](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ChainlinkOracle.sol#L49)

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ArbiChainlinkOracle.sol#L47](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ArbiChainlinkOracle.sol#L47)

### **Proof of concept**

ChainlinkOracle.sol:

```jsx
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

ArbiChainlinkOracle.sol:

```jsx
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

For both `getPrice()` and `getEthPrice()` the validation of the result when calling `latestRoundData` is insufficient. The code has a check if the returned price answer is non-negative, but this is insufficient, as there is other data returned from Chainlink’s oracle that should be properly checked to validate if the price is stale.

### **Impact**

If for some reason Chainlink data feed stops updating (it has happened before when Chainlink paused its $LUNA price feed for example) then the Risk Engine will mistakenly calculate an account’s total balance to be higher than it actually is. This can result in overleveraged borrowing from accounts even though they are unhealthy. 

## Recommendation

Change the `latestRoundData` logic to the following (you can use custom errors instead require statements as well):

```jsx
(roundId, rawPrice,, updatedAt, answeredInRound) = feedVariable.latestRoundData()
require(rawPrice > 0, "Chainlink price <= 0");
require(answeredInRound >= roundId, "Stale price");
require(block.timestamp - updatedAt < MAX_STALENESS_THRESHOLD_SECOND, "Stale price"); // You should decide the value of MAX_STALENESS_THRESHOLD_SECOND, for most cases 60 minutes is good enough
```

Change the code exactly the same way for all `latestRoundData` calls.