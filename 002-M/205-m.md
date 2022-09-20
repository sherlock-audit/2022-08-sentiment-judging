GalloDaSballo
# M-04 Chainlink Feed Price is not validated

## Summary

Chainlink Price Oracle doesn't check for recency, this can create situations in which the price feed isn't updated and the protocol will assume the price to be valid

## Vulnerability Detail

See the code:
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49

```solidity
        (, int answer,,,) =
            feed[token].latestRoundData();
```

No check is made to see the `updatedAt`

## Impact

In case of the FeedRegistry having any downtime, or network congestion, your protocol may end up using prices that are not updated, which can be exploited to leak value

## Tool used

Manual Review

## Recommendation

A simple check for `updatedAt` as shown in this code is sufficient:
https://github.com/GalloDaSballo/Super-Simple-Options/blob/7b817bb62be089116ff45502c370d2018d6cf62e/contracts/SuperSimpleCoveredCall.sol#L90

