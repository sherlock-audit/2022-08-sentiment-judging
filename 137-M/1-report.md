0x52
# Chainlink's latestRoundData may return stale or incorrect results

## Summary

Chainlink's latestRoundData may return stale or incorrect results

## Vulnerability Detail

## Impact

Incorrect of stale oracle results, leading to incorrect valuation of assets and potentially unfair liquidations

## Code Snippet

[ChainlinkOracle.sol#L49-L59](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L59)

[ChainlinkOracle.sol#L65-L73](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L65-L73)

## Tool used

Manual Review

## Recommendation

Round data should be check to confirm its validity as shown below:

    (, int answer,,,) =
        feed[token].latestRoundData();

    if (answer < 0)
        revert Errors.NegativePrice(token, address(feed[token]));

To this:

    (uint80 roundID, int256 answer, , uint256 timestamp, uint80 answeredInRound) = feed[token].latestRoundData();

    require(answeredInRound >= roundID, "Stale price");
    require(timestamp != 0,"Round not complete");
    require(answer > 0,"Chainlink answer reporting 0");