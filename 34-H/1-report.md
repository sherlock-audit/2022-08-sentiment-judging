grhkm
# AccountManager: address(0) used for borrowing ETH

## Summary
Using `address(0)` for borrowing ETH bricks an account.

## Vulnerability Detail
According to the following line in `settle`, `address(0)` is a valid value for `borrows` and used to indicate ETH:
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/AccountManager.sol#L322
While using `address(0)` in `borrow` (for `LEther`) would work, it would introduce different problems:
- `address(0)` would be added to the assets of the account, meaning that `RiskEngine._getBalance` would always revert because it would always call `IERC20(assets[i]).balanceOf(account)`
- A user could not call repay with `address(0)`, as this function reverts with this value in contrast to `borrow`
- Because of the above issue, `settle` would actually also not work (although it has a special case for `address(0)`), because it calls `repay` with `address(0)` in such situations.

## Impact
As mentioned above, using `address(0)` in `borrow` completely bricks an account. Of course, it is up to the sentiment team if this value is actually allowed (because when `LTokenFor(address(0))` is not set, the call to `borrow` fails). However, as mentioned, it looks like `address(0)` is intended to be added to. `LTokenFor`, otherwise there is no reason to handle it in `settle`.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/RiskEngine.sol#L157
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/AccountManager.sol#L236

## Tool used

Manual Review

## Recommendation
Disallow using `address(0)` in `borrow`, i.e. `require` that `token != address(0)`.