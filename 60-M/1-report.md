cergyk
# Oracle data feed is insufficiently validated

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ArbiChainlinkOracle.sol#L47-L59
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L73
## [M] Oracle data feed is insufficiently validated
### Problem
Oracle data feed is insufficiently validated. There is no check for stale price and round completeness.
Price can be stale and can lead to wrong prices returned
### Proof of Concept
```solidity
oracle/src/chainlink/ArbiChainlinkOracle.sol
47:     function getPrice(address token) external view override returns (uint) {
48:         if (!isSequencerActive()) revert Errors.L2SequencerUnavailable();
49: 
50:         (, int answer,,,) =
51:             feed[token].latestRoundData();
52: 
53:         if (answer < 0)
54:             revert Errors.NegativePrice(token, address(feed[token]));
55: 
56:         return (
57:             (uint(answer)*1e18)/getEthPrice()
58:         );
59:     }
```

```solidity
oracle/src/chainlink/ChainlinkOracle.sol
49:     function getPrice(address token) external view virtual returns (uint) {
50:         (, int answer,,,) =
51:             feed[token].latestRoundData();
52: 
53:         if (answer < 0)
54:             revert Errors.NegativePrice(token, address(feed[token]));
55: 
56:         return (
57:             (uint(answer)*1e18)/getEthPrice()
58:         );
59:     }

65:     function getEthPrice() internal view returns (uint) {
66:         (, int answer,,,) =
67:             ethUsdPriceFeed.latestRoundData();
68: 
69:         if (answer < 0)
70:             revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));
71: 
72:         return uint(answer);
73:     }
```

### Mitigation
```solidity
(uint80 roundID, int answer, uint startedAt, uint timeStamp, uint80 answeredInRound) = priceFeed.latestRoundData();

if (answer <= 0)revert Errors.ChainlinkInvalidPrice();
if (answeredInRound < roundID) revert Errors.ChainlinkStalePrice();
if (timestamp == 0 ) revert Errors.ChainlinkRoundNotComplete();

```