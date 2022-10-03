0xf15ers
# `TickMath` might revert in solidity `>=0.8.`

## Summary

## Vulnerability Detail
UniswapV3’s `TickMath` library was changed to allow compilations for solidity version `0.8`. However, adjustments to account for the implicit overflow behavior that the contract relies upon were not performed.

## Impact
In worst case, It could be that the library always reverts(instead of overflowing as in previous versions i.e `<0.8`) leading to broken contracts. 

## Code Snippet
- https://github.com/sherlock-audit/2022-08-sentiment-xremora/blob/main/oracle/src/uniswap/library/TickMath.sol

## Tool used

Manual Review

## Recommendation
- Follow the implementation of official [`TickMath 0.8`](https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/TickMath.sol) branch which uses [`unchecked`](https://github.com/Uniswap/v3-core/blob/6562c52e8f75f0c10f9deaf44861847585fc8129/contracts/libraries/TickMath.sol#L27) blocks for every function. 
