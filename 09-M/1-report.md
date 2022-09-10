cergyk
# LToken: Borrow rate manipulation possible

## Summary
There are situations where it is possible to manipulate the utilization in order to reduce the accrued interest.

## Vulnerability Detail
`getRateFactor()` passes `asset.balanceOf(address(this))` to the calculation of the borrow rate, which uses the value for the utilization calculation. Someone who has borrowed a lot (and therefore wants that the interest stays low), can increase this value before the interest rate accrual to keep the utilization rate low. While every call to `deposit` calls `updateState()` (which makes this attack not possible), it is possible to transfers assets into the contract before the call to `updateState()`. The assets will be lost, but it can still be beneficial if `updateState()` was not called in a long time (meaning that the reduction in accrued interest is large).

## Impact
Imagine the extreme case where Bob is the only borrower and the interest rate was not accrued for one year. Because utilization is very low, the accrued interest would be very high. Bob calculates that it is better for him to first transfer some assets to the LToken contract (that will be locked up forever) and then call `updateState()`. This results in a loss of interest for the borrowers.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/tokens/LToken.sol#L223

## Tool used

Manual Review

## Recommendation
Track the deposited assets in `ERC4626`, use that value instead of the balance.