grhkm
# ATokenOracle: aETH not correctly handled

## Summary
`UNDERLYING_ASSET_ADDRESS()` of aETH is not correctly handled.

## Vulnerability Detail
`UNDERLYING_ASSET_ADDRESS()` of aETH returns `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` (see https://docs.aave.com/developers/v/1.0/deployed-contracts/deployed-contract-instances or the actual contract: https://etherscan.io/address/0x3a3a65aab0dd2a17e3f1947ba16138cd37d08c04#readContract).
However, Sentiment handles the price of ETH by either requesting the value for `address(0)` or WETH.

## Impact
The ETH price will not be returned for aETH, the call will instead revert (meaning that aETH cannot be used).

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/aave/ATokenOracle.sol#L38

## Tool used

Manual Review

## Recommendation
Check if `UNDERLYING_ASSET_ADDRESS()` returns `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, request the price of `address(0)` in this case.