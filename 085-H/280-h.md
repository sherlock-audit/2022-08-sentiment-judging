GimelSec
# `LEther.sol` doesn't properly call `updateState()` upon receiving ether

## Summary

When doing `LEther.depositEth()` and `LEther.redeemEth()`, it won’t call `beforeDeposit()` and `beforeWithdraw()` to update the state of the lending pool.

## Vulnerability Detail

Normally, users call `LToken.deposit()` to deposit assets, and `LToken` will call `LToken.beforeDeposit()` which calls `LToken.updateState()` to update the state of the lending pool. However, `LEther.depositEth()` doesn’t call `beforeDeposit()`, therefore `updateState()` won’t be called. `LEther.redeemEth()` has the same problem. 

## Impact

LEther doesn’t update the state of the lending pool when calling `LEther.depositEth()` and `LEther.redeemEth()`. That is a serious issue.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/protocol/src/tokens/LEther.sol#L31-L38

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/protocol/src/tokens/LEther.sol#L47-L53

## Tool used

Manual Review

## Recommendation

Add `beforeDeposit()` in `depositEth()` and add `beforeWithdraw()` in `redeemEth()`

```diff
    function depositEth() external payable {
        uint assets = msg.value;
        uint shares = previewDeposit(assets);
        require(shares != 0, "ZERO_SHARES");
+       beforeDeposit(assets, shares);
        IWETH(address(asset)).deposit{value: assets}();
        _mint(msg.sender, shares);
        emit Deposit(msg.sender, msg.sender, assets, shares);
    }
```

```diff
    function redeemEth(uint shares) external {
        uint assets = previewRedeem(shares);
        _burn(msg.sender, shares);
+       beforeWithdraw(assets, shares);
        emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
        IWETH(address(asset)).withdraw(assets);
        msg.sender.safeTransferEth(assets);
    }
```
