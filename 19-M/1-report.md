grhkm
# ChainlinkOracle: Assumes all feeds have 8 decimals

## Summary
There is a wrong assumption about the feed decimals in `ChainlinkOracle` which causes it to return wrong prices.
Edit: Just saw in the previous audit report that this was already documented and the team plans to only use feeds with 8 decimal places. Still seems quite risky to me, because the system will be broken if one feed is chosen with more decimals (and it is not validated that they have 8 decimals), but downgrading to Medium because of that.

## Vulnerability Detail
The answer of the chainlink oracle is multiplied by 10^18 and then divided by the ETH/USD price feed. In `_valueInWei` within `RiskOracle`, this price is then multiplied by the token amount and divided by 10^decimals to get the value in wei (18 decimals). While this would be correct when all chainlink feeds would have the same number of decimals, this is not the case. See for instance https://ethereum.stackexchange.com/questions/92508/do-all-chainlink-feeds-return-prices-with-8-decimals-of-precision for a discussion of this. Non-ETH pairs (such as the ETH / USD feed that is used) have 8 decimals, whereas ETH pairs have 18.

## Impact
The price that is returned by `getPrice(token)` has 28 decimals (18 + 18 - 8), leading to completely wrong prices. This can be exploited to open lending positions where the health factor (if it would be calculated correctly) is way below 1.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/chainlink/ChainlinkOracle.sol#L57
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/RiskEngine.sol#L186

## Tool used

Manual Review

## Recommendation
Normalize the price to 18 decimals.