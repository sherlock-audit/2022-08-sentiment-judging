minera
# The chainlink oracle data to determine the collateral worth may be outdated because a invalid timestamp is used to check if the oracle data is up-to-date. 

## Summary

The chainlink oracle data to determine the collateral worth may be outdated because a invalid timestamp is used to check if the oracle data is up-to-date in ArbiChainlinkOracle.sol.


## Vulnerability Detail

in ArbiChainlinkOracle.sol

the function getPrice is implemented to make sure we get the underlying token worth when calculating how much the collateral is worth.

```
    /// @inheritdoc ChainlinkOracle
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

note we check if the oracle data is up-to-date in the

```
    function isSequencerActive() internal view returns (bool) {
        (, int256 answer, uint256 startedAt,,) = sequencer.latestRoundData();
        if (block.timestamp - startedAt <= GRACE_PERIOD_TIME || answer == 1)
            return false;
        return true;
    }
```

according to the chainlink oracle documentation

https://docs.chain.link/docs/price-feeds-api-reference/#latestrounddata

the lastestRoundData returns

```
roundId: The round ID.
answer: The price.
startedAt: Timestamp of when the round started.
updatedAt: Timestamp of when the round was updated.
answeredInRound: The round ID of the round in which the answer was computed.
```

the current implementation use the startedAt timestamp instead of updatedAt to check if the oracle is valid.

In this case when the oracle data is lagging, it is possible that we have the huge time difference between the startedAt and updatedAt timestamp.

Also, the grace period is 1 hour. which is very long because the crypto market prices can change drastically in a volatile period.  So the oracle data from an hour ago may not reflect the current market price.

```
    /// @notice L2 Sequencer grace period
    uint256 private constant GRACE_PERIOD_TIME = 3600;
```


## Impact

```
  if (block.timestamp - startedAt <= GRACE_PERIOD_TIME || answer == 1)
            return false;
```


If the oracle is lagging, the logic above is not capable to invalidating outdated oracle data because the use of startedAt and a long grace period. then the function _valueIn can get the invalid or lagging oracle and determine the wrong number of debt or callateral worth. User may not able to deposit or repay to manage their position or malicious liquidators can liquidated user's account balance.

```
    function _valueInWei(address token, uint amt)
        internal
        view
        returns (uint)
    {
        return oracle.getPrice(token)
        .mulDivDown(
            amt,
            10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
        );
    }
```

## Tool used

Manual Review

Yes

## Recommendation

We recomment the project to use updatedAt timestamp and shorten the grace period to make sure the oracle data is up to date. 

Fix:

```
    /// @notice L2 Sequencer grace period
    uint256 private constant GRACE_PERIOD_TIME = 500;
```

```
    function isSequencerActive() internal view returns (bool) {
        (, int256 answer,, updatedAt,) = sequencer.latestRoundData();
        if (block.timestamp - updatedAt <= GRACE_PERIOD_TIME)
            return false;
        return true;
    }
```
