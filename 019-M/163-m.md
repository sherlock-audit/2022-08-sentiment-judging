pashov
# Price decimals assumptions in `ChainlinkOracle.sol` & `ArbiChainlinkOracle.sol` can lead to incorrect calculation of price

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ChainlinkOracle.sol#L57](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ChainlinkOracle.sol#L57)

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ArbiChainlinkOracle.sol#L57](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/oracle/src/chainlink/ArbiChainlinkOracle.sol#L57)

## Proof of Concept

In both **`ChainlinkOracle.sol` & `ArbiChainlinkOracle.sol`** when we calculate the price of an asset we have the following code:

```jsx
(uint(answer)*1e18)/getEthPrice()
```

where `answer` is coming from `(, int answer,,,) = feed[token].latestRoundData()`

This assumes that the `token` price feed has the same decimals as the ETH price feed (8). This is also mentioned in a comment in the NatSpec 

```
/// @dev feed[token].latestRoundData should return price scaled by 8 decimals
```

This is a common issue where code makes assumptions and even tries guiding devs with comments, but the vulnerability is still there even though it can be easily mitigated. Right now nothing is stopping an admin (unintentionally or maliciously) to add a token which has a price feed with less or more than 8 decimals (Chainlink has such price feeds [https://docs.chain.link/docs/ethereum-addresses/](https://docs.chain.link/docs/ethereum-addresses/)) and if this happens then the price will be calculated very wrong.

## Impact

If the price of an asset is too small it can lead to unexpected liquidations. If the price of an asset is too big it can lead to overleveraged borrowing. Both of those can result in a loss of funds for the protocol or the users.

## Recommendation

Enforce the NatSpec comment that each price feed added should have 8 decimals by just adding the following check in `setFeed()` 

```jsx
require(_feed.decimals() == 8, "Only 8 decimal feeds allowed");
```

With this you can remove all assumptions and let the code work in a trustless fashion.