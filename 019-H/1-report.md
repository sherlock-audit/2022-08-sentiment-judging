WATCHPUG
# `ChainlinkOracle.sol#getPrice()` The price will be wrong when the token's USD price feed's `decimals != 8`

## Summary

`ChainlinkOracle` assumes and inexplicitly requires the token's USD feed's decimals to be 8. However, there are certain token's USD feed has a different `decimals`.

## Vulnerability Detail

In the current implementation, it assumes $tokenFeedDecimals = ethFeedDecimals$ (`feed[token].decimals()` must equals `ethUsdPriceFeed.decimals() = 8`).

However, there are tokens with USD price feed's decimals != 8 (E.g: [`AMPL / USD` feed decimals = 18](https://etherscan.io/address/0xe20CA8D7546932360e37E9D72c1a47334af57706#readContract))

When the token's USD feed's decimals != 8, `ChainlinkOracle.sol#getPrice()` will return an incorrect price in ETH.

The correct calculation formula should be:

$$
\frac{ answer_{token} \cdot 10^{ethFeedDecimals} }{ answer_{eth} \cdot 10^{tokenFeedDecimals} } \cdot 10^{18}
$$

### PoC

Given:

-   1.0 AMPL worth 1.14 USD, `feed[ampl].decimals() == 18`, `answer_ampl = 1140608758261546000` [Source: `feed[ampl]`](https://etherscan.io/address/0xe20CA8D7546932360e37E9D72c1a47334af57706#readContract)

-   1.0 ETH worth 1588.11 USD, `ethUsdPriceFeed.decimals() == 8`, `answer_eth = 158811562094` [Source: `ethUsdPriceFeed`](https://etherscan.io/address/0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419#readContract)

`chainlinkOracle.getPrice(AMPL)` will return ~7.18m (eth):

$$
\frac{ answer_{token} \cdot 10^{18} }{ answer_{eth} } = \frac{ 1140608758261546000 \cdot 10^{18} }{ 158811562094 } = 7182151873718260663354494
$$

## Impact

When the price feed with `decimals != 18` is set, the attacker can deposit a small amount of the asset and drain all the funds from the protocol.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L178-L188

```solidity
function _valueInWei(address token, uint amt)
    internal
    view
    returns (uint)
{
    return oracle.getPrice(token)
    .mulDivDown(
        amt,
        10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
    );
}
```

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L47-L59

```solidity
/// @inheritdoc IOracle
/// @dev feed[token].latestRoundData should return price scaled by 8 decimals
function getPrice(address token) external view virtual returns (uint) {
    (, int answer,,,) =
        feed[token].latestRoundData();

    if (answer < 0)
        revert Errors.NegativePrice(token, address(feed[token]));

    return (
        (uint(answer)*1e18)/getEthPrice()
    );
}
```


https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L65-L73

```solidity
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

## Recommendation

Consider adding a check for `feed.decimals()` to make sure `feed`'s decimals = `8`:

```solidity
constructor(AggregatorV3Interface _ethUsdPriceFeed) Ownable(msg.sender) {
    require(_ethUsdPriceFeed.decimals() == 8, "...");
    ethUsdPriceFeed = _ethUsdPriceFeed;
}
```

```solidity
function setFeed(
    address token,
    AggregatorV3Interface _feed
) external adminOnly {
    require(_feed.decimals() == 8, "...");
    feed[token] = _feed;
    emit UpdateFeed(token, address(_feed));
}
```
