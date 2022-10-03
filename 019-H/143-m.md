berndartmueller
# Chainlink oracle calculates the wrong price for ETH nominated pairs

## Summary

Chainlink feed pairs nominated in ETH are higher valued than their actual value.

## Vulnerability Detail

Chainlink feed pairs have 8 decimals unless it's an ETH-nominated pair (for instance, `LDO/ETH` is nominated in `ETH` and has 18 decimals). The current implementation of `ChainlinkOracle.getPrice` expects the returned price to have 8 decimals. The calculation works as intended for pairs with 8 decimals, and the price returned by the `getPrice` function has 18 decimals.

However, if the `token` is nominated in ETH, the Chainlink oracle feed will return the price scaled by **18 decimals**. Hence the calculation of `getPrice` will return a price scaled to **28 decimals** (off by 10 decimals).

For reference, see the [list of Oracle feeds](https://docs.chain.link/docs/ethereum-addresses/) and their decimals

## Impact

Suppose the protocol has a token allowlisted (e.g. `LIDO`) that is nominated in `ETH`. In that case, the user's account balance is inflated and the user can borrow and withdraw more than the "real" value of the deposited collateral.

## Code Snippet

[oracle/src/chainlink/ChainlinkOracle.sol#L49-L73](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L73)

```solidity
function getPrice(address token) external view virtual returns (uint) {
    (, int answer,,,) =
        feed[token].latestRoundData();

    if (answer < 0)
        revert Errors.NegativePrice(token, address(feed[token]));

    return (
        (uint(answer)*1e18)/getEthPrice() // @audit-info Always multiplying by 1e18 will lead to wrong calculations if the token is nominated in ETH
    );
}

function getEthPrice() internal view returns (uint) {
    (, int answer,,,) =
        ethUsdPriceFeed.latestRoundData();

    if (answer < 0)
        revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

    return uint(answer);
}
```

## Tools Used

Manual review

## Recommendation

Either make sure only to use USD nominated Chainlink feeds or check the number of decimals of the used Chainlink feed (`feed[token].decimals()`) and calculate the price accordingly.
