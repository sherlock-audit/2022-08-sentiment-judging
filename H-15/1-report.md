0xNineDec
# Oracle data feed may return stale prices for tokens and ether

## Summary

Currently there is a heavy dependency on the retrieved oracle prices for both ether and tokens. There is no freshness check of the retrieved data thus stale prices could be used anyways.

## Vulnerability Detail

The data feeds that the `ChainlinkOracle` implementation provides are heavily used and consulted by the protocol. The freshness and reliability of data feeds and prices across a yield protocol is crucial. The protocol does not check the last update time of the oracle retrieved prices and will use them across the protocol.

## Impact

The implementations that consult to Chainlink's oracle, only perform checks for non-negative values but no checks are performed in relationship to the freshness of the retrieved data. If the oracle fails or the data is stale, the protocol will use it anyways impacting directly to lenders, borrowers and other concerned parties.

## Code Snippet

Currently the oracles implementation are the following:

```solidity
ChainlinkOracle.getPrice()

function getPrice(address token) external view virtual returns (uint) {
    (, int answer,,,) =
        feed[token].latestRoundData();

    if (answer < 0)
        revert Errors.NegativePrice(token, address(feed[token]));

    return (
        (uint(answer)*1e18)/getEthPrice()
    );
}

ChainlinkOracle.getEthPrice()

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

As Chainlink [recommends](https://docs.chain.link/docs/using-chainlink-reference-contracts/#check-the-timestamp-of-the-latest-answer):

> Your application should track the `latestTimestamp` variable or use the `updatedAt` value from the `latestRoundData()` function to make sure that the latest answer is recent enough for your application to use it. If your application detects that the reported answer is not updated within the heartbeat or within time limits that you determine are acceptable for your application, pause operation or switch to an alternate operation mode while identifying the cause of the delay.

> During periods of low volatility, the heartbeat triggers updates to the latest answer. Some heartbeats are configured to last several hours, so your application should check the timestamp and verify that the latest answer is recent enough for your application.

It is recommended both to add also a tolerance that compares the `updatedAt` return timestamp from `latestRoundData()` with the current block timestamp and ensure that the `priceFeed` is being updated with the required frequency.

