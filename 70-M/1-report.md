CRYP70
# ERC20 Tokens with Different Decimals to 1e18 lead to Inaccurate Price Feed

## Summary
There exists an issue in the Oracle contracts in the function(s) `getPrice(address token)` where given a token, will return the latest price. These prices are assumed to be 18 decimal points which may lead to an inaccurate reading if an alternative token was used.  

## Vulnerability Analysis 
Given the function `getPrice(address token)` taken from the `ChainlinkOracle.sol` contract which attempts to retrieve the latest price through `getEthPrice()` which utilises `ethUsdPriceFeed`:
```solidity

    /// @inheritdoc IOracle
    /// @dev feed[token].latestRoundData should return price scaled by 8 decimals
    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()  // <---------
        );
    }
```

If an alternative token was supplied to the function such as `USDC` or `USDT`,  an inaccurate reading could be returned because these tokens for example are of `6` decimal places as opposed to `18`. 

## Impact
This affects the following oracle contracts:
- `ArbiChainlinkOracle.sol`:`57`
- `ChainlinkOracle.sol`:`57`
- `CurveTriCryptoOracle.sol`:`52`

## Recommendations
I recommend adding some support for a variety of decimal places as it varies from token to token by dynamically factoring `IERC20(token).decimals()` into the calculations for determining token prices. 
