grhkm
# LEther: Flash loan can be used to manipulate borrow rate

## Summary
`LEther` mistakenly does not call `beforeDeposit` before a deposit happens, which can be exploited to reduce interest significantly.

## Vulnerability Detail
In `depositEth` within `LEther`, `beforeDeposit` which updates the state is not called. Because of that, the utilization can be manipulated within a single transaction. As this is unintended behavior, there are potentially more ways to exploit it (combination of borrowing, depositing, withdrawing, etc...), but one way how it can be exploited to reduce interest significantly is documented below.

## Impact
Someone who has borrowed money (and therefore wants to reduce interest) takes out a huge ETH flash loan and deposits the ETH via `depositEth()`. Then, he calls `updateState()` which now uses the significantly increased (W)ETH balance. Therefore, utilization will be almost 0 and almost no interest will be accrued. Afterwards, he calls `redeemEth()` and pays back the flashloan.

Note that this attack is most effective when `updateState()` was not called for a long time. However, it can also be performed multiple times (e.g., front-running all transactions that would update the state) such that interest is never properly accrued.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/tokens/LEther.sol#L36

## Tool used

Manual Review

## Recommendation
Call `beforeDeposit()` in `depositEth()` (and `beforeWithdraw()` in `redeemEth()`).