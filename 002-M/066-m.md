Ruhum
# Chainlink oracle isn't validated properly

## Summary
Chainlink oracles have to be validated for the data's recency. Otherwise, your protocol might work with stale data.

## Vulnerability Detail
A recent example is the crash of LUNA and UST. The LUNA Chainlink oracle stopped updating at a price of `$0.10` although the actual price of the asset was way lower. That allowed attackers to borrow more assets for their deposited LUNA tokens than they should be on the Blizz protocol, see [their post mortem](https://medium.com/@blizzfinance/blizz-finance-post-mortem-2425a33fe28b).
The Sentiment protocol doesn't validate the oracle at all which makes them vulnerable to the same issue as Blizz.

## Impact
An attacker will be able to borrow more funds than they should be.

## Code Snippet
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L50-L58

```sol
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

The price is used to determine the value in WEI which is used to determine the state of the account: https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L183

## Tool used

Manual Review

## Recommendation
Chainlink recommends the following steps for risk mitigation: https://docs.chain.link/docs/selecting-data-feeds/#risk-mitigation