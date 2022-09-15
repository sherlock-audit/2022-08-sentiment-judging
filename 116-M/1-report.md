rajatbeladiya
# Repay loan will throw transaction underflow error

## Summary
if repay amount is greater than loan amount, transaction will reverted with underflow error. 

## Vulnerability Detail
1. Alice borrow 100 usdt.
2. Alice repay using `repay()` function with 110 usdt.
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L227-L242
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L237
3. It will redirect to `collectFrom()` function where it subtract `repay amount` from `borrows`.
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L153-L165
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L163

## Impact
`repay()` function will not work for greater amount than borrows.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L153-L165


## Tool used
Manual Review

## Recommendation
add `require(amt <= borrows, "repay amount should less than borrows")` or `subtract only borrow amount and return extra amount`