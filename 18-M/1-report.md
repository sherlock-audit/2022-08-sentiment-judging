grhkm
# ChainlinkOracle: Might return stale values from previous round

## Summary
The `roundId` in `ChainlinkOracle` is not validated.

## Vulnerability Detail
According to the [Chainlink Docs](https://docs.chain.link/docs/historical-price-data/), "You can check answeredInRound against the current roundId. If answeredInRound is less than roundId, the answer is being carried over. If answeredInRound is equal to roundId, then the answer is fresh."
However, this is not done.

See also https://github.com/code-423n4/2022-04-backd-findings/issues/17 for a discussion of the issue.

## Impact
The Oracle might return stale prices. This can be exploited by an attacker that buys the token (that has now a lower price), uses it as collateral, and takes out loans. His real health ratio would be below 1, but this is not detected because of the stale prices.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/chainlink/ChainlinkOracle.sol#L66

## Tool used

Manual Review

## Recommendation
Check that `answeredInRound >= roundID`.