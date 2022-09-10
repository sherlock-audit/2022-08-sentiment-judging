cergyk
# CTokenOracle: Wrong price for CEther

## Summary
The value that is returned by `CTokenOracle` for cETH leads to wrong exchange calculations.

## Vulnerability Detail
The exchange rate that is returned by `exchangeRateStored()` should not be multiplied by 1e18, this leads to wrong values.

## Impact

For instance, at the time of writing, 1 cETH = 0.020067 ETH and `exchangeRateStored` returns 200674139044261374629937592. cETH has 8 decimals (see https://etherscan.io/token/0x4ddc2d193948926d02f9b1fe9e1daa0718270ed5#readContract), but the system would return the following for `_valueInWei(address(cETH), 1e8)`:
```
((200674139044261374629937592 * 1e8) * 1e8) / 10 ** IERC20(cETH).decimals() =
((200674139044261374629937592 * 1e8) * 1e8) / 10 ** 8 =
~2e34
```
Which is completely wrong.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/compound/CTokenOracle.sol#L63

## Tool used

Manual Review

## Recommendation
Replace `getCEtherPrice()` with
```
return ICToken(cETHER).exchangeRateStored().divWadDown(1e10);
```
Then, the above calculation returns ~2e16, which is correct (1 cETH is currently worth roughly 2e16 wei, i.e. 0.02 ETH).