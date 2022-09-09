minera
# isSequencerActive() uses startedAt instead of updatedAt

## Summary
`isSequencerActive()` determines whether the sequencer is active by checking `startedAt` instead of `updatedAt`.

## Vulnerability Detail
The `isSequencerActive` function incorrectly uses `startedAt` to determine whether the data reported by Chainlink oracle is within grace period. Instead, it should use `updatedAt` which represents the timestamp of when the round was updated, not the timestamp of when the round started.

## Impact
`isSequencerActive` will return false even though the data provided by Chainlink oracle is valid and fresh. 

## Code Snippet

```solidity
// oracle/src/chainlink/ArbiChainlinkOracle.sol:65
function isSequencerActive() internal view returns (bool) {
        (, int256 answer, uint256 startedAt,,) = sequencer.latestRoundData();
        if (block.timestamp - startedAt <= GRACE_PERIOD_TIME || answer == 1)
            return false;
        return true;
    }
```
## Tool used

Manual Review
https://docs.chain.link/docs/price-feeds-api-reference/

## Recommendation
Use `updatedAt` instead.

```solidity
function isSequencerActive() internal view returns (bool) {
        (, int256 answer,, uint256 updatedAt,) = sequencer.latestRoundData();
        if (block.timestamp - updatedAt <= GRACE_PERIOD_TIME || answer == 1)
            return false;
        return true;
    }
```