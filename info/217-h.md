__141345__
# Liquidation incentive 

## Summary

Currently the `balanceToBorrowThreshold` is set to 1.2, however if the `balance/borrow` ratio drop below 1.0 (such as severe market conditions, contract DoS, omitted accounts, etc), the liquidator won't have the incentive to repay the loan, since the collateral received worth less than the repay amount. In such situation, the liquidator won't help to keep the protocol solvent as intended. And the protocol will continue to lose more as no liquidator will do the work.



## Vulnerability Detail

Liquidations are not guaranteed to be triggered when the `balance/borrow` ratio hit 1.2, such as:
- severe market conditions: the market plunges so that liquidators can not timely take reactions
- contract DoS: if the contract paused for a period, it is possible for some account to be deep underwater
- omitted accounts: some account just missed by chance

When it drops to a certain value, maybe barely larger than 1, the liquidation won't be profitable. As a results, liquidation mechanism will stop working to keep the protocol solvent. Liquidators repay the loan but receive less than what they paid for. As no liquidation is performed, the loss might keep on growing.

Especially during market crash, the loss would be more disastrous, since the collateral might drop to zero, like UST/luna case. If liquidation cannot function timely and just stop working, the loss will be out of control.


## Impact

According to the currently incentive setup, if the `balance/borrow` ratio drop to some unexpected value, the liquidation mechanism will stop working. In certain situations, the protocol loss will keep growing, go out of control. 




## Code Snippet

`balanceToBorrowThreshold` is set to be 1.2.
```solidity
// protocol\src\core\RiskEngine.sol

    uint public constant balanceToBorrowThreshold = 1.2e18;

    function _isAccountHealthy(uint accountBalance, uint accountBorrows)
        internal
        pure
        returns (bool)
    {
        return (accountBorrows == 0) ? true :
            (accountBalance.divWadDown(accountBorrows) > balanceToBorrowThreshold);
    }
```



## Tool used

Manual Review

## Recommendation

If the `balance/borrow` ratio drop to a certain value, consider add other incentives for the liquidators, such as additional discount, or other rewards. Just to make sure it will always be profitable for liquidators, then the liquidation mechanism can be more robust to keep the protocol solvent.
