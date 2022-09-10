cergyk
# Insufficient validation in Oracle data feed

## Summary
When fetching prices from `latestRoundData`, there is not enough validation that ensures the price returned is not stale.

## Vulnerability Detail
The `getPrice` and `getEthPrice` function fetches oracle price from Chainlink using `latestRoundData`, there only exist one check to make sure the reported price is not negative value. As prices can never be negative value, this check is not needed. Instead, there should be sufficient checks to make sure Chainlink's reported price feed is fresh and not stale.

https://docs.chain.link/docs/historical-price-data/#historical-rounds
https://consensys.net/diligence/audits/2021/09/fei-protocol-v2-phase-1/#chainlinkoraclewrapper---latestrounddata-might-return-stale-results

## Impact
Stale prices might be used.

## Code Snippet
- oracle/src/chainlink/ArbiChainlinkOracle.sol:51
- oracle/src/chainlink/ChainlinkOracle.sol:51
- oracle/src/chainlink/ChainlinkOracle.sol:67

```solidity
// oracle/src/chainlink/ArbiChainlinkOracle.sol:51
function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }

// oracle/src/chainlink/ChainlinkOracle.sol:51
function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }

// oracle/src/chainlink/ChainlinkOracle.sol:67
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
[Chainlink's deployed code](https://etherscan.io/address/0x986b5E1e1755e3C2440e960477f25201B0a8bbD4#code)

## Recommendation

Validate data feed by:
- Checking the returned `answer` is not 0.
- Verify result is within an allowed margin of freshness by checking `updatedAt`.
- Verify `answer` is indeed for the last known round. 

```solidity
function getEthPrice() internal view returns (uint) {
        (uint80 ethRoundID, int answer,, uint256 ethUpdatedAt, uint80 ethAnsweredinRound) =
            ethUsdPriceFeed.latestRoundData();
        require(ethUpdatedAt > block.timestamp - 3600, "ETH: Stale price");
        require(ethAnsweredinRound == ethRoundID , "ETH: Answer is not for last known round!");
        if (answer <= 0)
            revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

        return uint(answer);
    }
```

