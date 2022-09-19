GimelSec
# Price oracle could get a stale price

## Summary

`getPrice` in ArbiChainlinkOracle.sol and ChainlinkOracle.sol may get stale price.

## Vulnerability Detail

There is no check for the stale price of `answer`, `updateAt` and `roundId`.
https://github.com/code-423n4/2022-01-yield-findings/issues/136

## Impact

Price oracle could get a stale price without checking `roundId`.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/oracle/src/chainlink/ArbiChainlinkOracle.sol#L50-L51

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/oracle/src/chainlink/ChainlinkOracle.sol#L50-L51

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/oracle/src/chainlink/ChainlinkOracle.sol#L66-L67

## Tool used

Manual Review

## Recommendation

Check `answer`, `updateAt` and `roundId` when getting price:

```
        (uint80 roundId, int256 answer, , uint256 updatedAt, uint80 answeredInRound) = oracle.latestRoundData();

        require(updatedAt > 0, "Round is not complete");
        require(answer >= 0, "Malfunction");
        require(answeredInRound >= roundID, "Stale price");
```
