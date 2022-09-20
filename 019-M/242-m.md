Bahurum
# Missing check on decimals of Chainlink price feed

## Summary
In `ChainlinkOracle.getPrice()` the number of decimals of the price feed is not verified. In case a feed with decimals different than 8 is added in the future by the admin to the `feed` mapping, the price calculated by `getPrice()` would be badly off.

## Vulnerability Detail
Currently all chainlink feeds of USD pairs have a precision of 8 decimals, but it is possible that this could change in the future. The admin could add in the future a feed with decimals > 8 without considering this risk. The contract assumes that the price feed returns 8 decimals and `getPrice()` succeeds returning a false value if it is not the case.

## Impact
If the decimals of the price feed are > 8, then the price is overestimated by orders of magnitude, and the collateral corresponding to `token` is overestimated accordingly, letting any account that owns `token` borrow and then withdraw all the liquidity from the protocol.

## Code Snippet

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L59

```solidity
   function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();  // @audit feed must be eth for token == address(0)

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```

## Tool used

Manual Review

## Recommendation
Check that the decimals of the feed is 8 and revert otherwise, 
``` solidity 
require(feed[token].decimals() == 8 , "Feed has not 8 decimals")
```
or, alternatively, get the decimals for both `ethUsdPriceFeed` feed and `feed[token]`, and normalise the price feed:

``` diff
    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();
+       uint8 decimals = feed[token].decimals();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

+       (uint256 ethUsdPrice, uint8 ethUsdDecimals) = getEthPrice()
        return (
-           (uint(answer)*1e18)/getEthPrice()
+           (uint(answer)*1e18*10**(ethUsdDecimals))/(ethUsdPrice*10**(decimals))
        );
    }


    function getEthPrice() internal view returns (uint) {
        (, int answer,,,) =
            ethUsdPriceFeed.latestRoundData();
+       uint8 decimals = ethUsdPriceFeed.decimals();

        if (answer < 0)
            revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));
-       return uint(answer)
+       return (uint(answer), decimals);
    }
```