pashov
# Some functionalities in `LEther.sol` & `LToken.sol` are not calling `beforeDeposit` and `beforeWithdraw` hooks

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LEther.sol#L31](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LEther.sol#L31)

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LEther.sol#L47](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LEther.sol#L47)

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LToken.sol#L153](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LToken.sol#L153)

### Proof of concept

In `ERC4626.sol` you have 4 methods that move assets to/from the vault - `deposit`, `mint`, `withdraw`, `redeem`. All of them use either the `beforeDeposit` or `beforeWithdraw` hooks which in `LToken.sol` are implemented to call `updateState()`. Calling `updateState()` before/after moving assets to/from the vault is important as it is the accounting logic behind the interestAccrued in `borrows` and `reserves`. The problem is that in `depositEth()` and `redeemEth()` in `LEther.sol` we do not call any hooks - the methods goes straight to calling `previewDeposit()` and `previewRedeem()`. The `updateState()` call is also missing from LToken's `collectFrom()` method, even though it is there for `lendTo()`.

### Impact

Not calling `updateState()` when there is value transfer can lead to inaccurate calculations of shares when depositing ETH through `LEther.sol` and inaccurate calculations of assets when redeeming ETH through `LEther.sol` . This can result in the user depositing/redeeming ETH permanently losing ETH value due to those inaccurate calculations.

### Recommendation

In `LEther.sol` add `beforeDeposit()` hook in the first line of `depositEth()` and add `beforeWithdraw()` in the first line of `redeemEth()`. Also add `updateState()` to the first line of `collectFrom()` in `LToken.sol`