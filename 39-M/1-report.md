grhkm
# Decimals overflow can happen in getprice function

## Summary
tokens with decimals higher than 18 will always revert.

## Vulnerability Detail
L55 will revert when token has higher than 18 decimals.



## Impact
This will cause inabilities for the Getprice function 


## Code Snippet
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/uniswap/UniV3TWAPOracle.sol#L55
## Tool used

Manual Review

## Recommendation
Consider modifying how GetPrice func to could handle tokens with higher than 18 decimals.


