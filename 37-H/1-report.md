cergyk
# Oracle Flashloans attacks to force liquidations possible

## Summary

## Vulnerability Detail
Some of the used oracles can be manipulated with flashloans:

- The curve tricrypto pool only has a market cap of $399.3k at the time of writing. Therefore, the WETH price that is directly retrieved from the pool (`pool.price_oracle(1)`) could easily be manipulated with a flashloan.
- yearns `pricePerShare()` is manipulable by flashloans, which was exploited previously: https://medium.com/cream-finance/post-mortem-exploit-oct-27-507b12bb6f8e
- Compounds `exchangeRateStored()` directly queries balances and is manipulable (see e.g. the cETH code: https://etherscan.io/token/0x4ddc2d193948926d02f9b1fe9e1daa0718270ed5#code)

## Impact
An attacker can use flashloans to force liquidations of users that hold these assets. They will be liquidated, although their health ratio (if properly calculated) would be significantly higher.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/curve/CurveTriCryptoOracle.sol#L52

## Tool used

Manual Review

## Recommendation
Use flashloan-resistant oracles.