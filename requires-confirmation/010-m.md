Lambda
# LToken: redeemReserves does not update borrows

## Summary
Accounting error when redeeming reserves.

## Vulnerability Detail
In `updateState()`, `borrows` is increased by the accrued interest including reserves. However, when reserves are redeemed, `borrows` is not decreased by the corresponding amount. Therefore, `totalAssets()` will return a wrong value after redemption, because the redeemed reserve amount is still included in `getBorrows()`, but no longer subtracted (as it is no longer included in `getReserves()`).

## Impact
After redemption, `convertToShares` and `convertToAssets` no longer work correctly, as they call `totalAssets()` internally.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/tokens/LToken.sol#L248

## Tool used

Manual Review

## Recommendation
Decrease `borrows` by the redeemed amount.


## Sherlock Comment

The `borrows` accounting seems correct as the `redeemReserves` only affects the tokens that are not borrowed, so a decrease on `borrows` is not needed.