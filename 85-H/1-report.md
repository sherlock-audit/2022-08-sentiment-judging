bytehat
# LEther interest rate can be manipulated to be essentially zero

## Summary
Anyone can manipulate the interest rate of LEther to be close to zero. Since `depositEth` and `redeemEth` functions do not call `updateState` in any way, a malicious user can `flashloan eth -> depositEth -> updateState -> redeemEth -> pay back flashloan` in one transaction and zero out the incremental interest that should have been accrued.

## Vulnerability Detail
In the LToken contract, the interest rate is updated periodically by the public `updateState` function. Important functions like `deposit` and `lendTo` make sure to first call `updateState` (either explicitly or through the `beforeDeposit` and `beforeWithdraw` hooks) so that the accrued interest is properly handled. The LEther contract adds the `depositEth` and `redeemEth` functions as well, which do not call `updateState` in any way. This allows anyone to manipulate the interest rate to be close to zero in the following way:

1. An attacker takes out a large flashloan of eth. There are many flashloan providers today that require no fee - for example Euler finance has 27 million USD worth of WETH available for flashloan using the contract here: https://etherscan.io/address/0x07df2ad9878F8797B4055230bbAE5C808b8259b3 (this doesn't include all the other tokens you can flashloan for free from Euler).
2. The attacker calls `depositEth`. Since the attacker could have flashloaned a huge amount of eth for zero cost, the utilization ratio of LEther will be close to zero based on the rate model. This means `getBorrowRatePerSecond` will return a value very close to zero.
3. The attacker calls `updateState`. Since the utilization ratio is essentially zero, the incremental interest accrued since last update will also be essentially zero, and the `lastUpdated` timestamp will be updated.
4. The attacker calls `redeemEth` and pays back their flashloans for no fee.

This benefits the borrowers since they do not have to pay any interest on their borrowed funds. This harms the lenders (and the protocol) since they lose out on the payment they should have received for taking on smart contract risk and default risk. An attacker could even write a wrapper contract for borrowers on Sentiment that always does this attack before calling the main protocol, so that interest rates are always zero. An attacker can also front-run legitimate calls to `updateState` to execute this attack.

## Impact
Borrowers can essentially steal the interest accrued that should be given to the lenders and protocol.

## Code Snippet
Notice no call to `updateState` here:
```
function depositEth() external payable {
    uint assets = msg.value;
    uint shares = previewDeposit(assets);
    require(shares != 0, "ZERO_SHARES");
    IWETH(address(asset)).deposit{value: assets}();
    _mint(msg.sender, shares);
    emit Deposit(msg.sender, msg.sender, assets, shares);
}
```

## Tool used
Manual Review

## Recommendation
Add a call to `beforeDeposit` in `depositEth` and a call to `beforeWithdraw` in `redeemEth`.