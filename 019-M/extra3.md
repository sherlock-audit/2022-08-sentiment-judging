sorrynotsorry
# Chainlink price decimals are assumed as 18

## Summary

Chainlink price decimals are assumed as 18

## Vulnerability Detail

This may cause erroneous math and book keeping.

## Impact

The response from the price oracle always assumes 18 decimals in `(uint(answer)*1e18)/getEthPrice()` but it's never checked if the oracle response has 18 decimals using ChainLink's .decimals() function.

## Code Snippet

```solidity
57: (uint(answer)*1e18)/getEthPrice()
```

[Permalink](https://github.com/sherlock-audit/2022-08-sentiment-0xsorrynotsorry/blob/9d3693f0aff44a71638070e8794e83a16867bfca/oracle/src/chainlink/ChainlinkOracle.sol#L57)

## Tool used

Manual Review

## Recommendation

Consider using `decimals()` function.
