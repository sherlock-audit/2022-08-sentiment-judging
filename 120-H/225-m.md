cccz
# AccountManager: updateState() is not called before calling isWithdrawAllowed()/isBorrowAllowed()/isAccountHealthy() of the riskEngine contract

## Summary
AccountManager: updateState() is not called before calling isWithdrawAllowed()/isBorrowAllowed()/isAccountHealthy() of the riskEngine contract.
## Vulnerability Detail
isWithdrawAllowed()/isBorrowAllowed()/isAccountHealthy() of the riskEngine contract will use the result of _getBorrows(), which is affected by the borrows variable in the LToken contract.
So call the LToken contract's updateState() to use the latest borrows variable before calling the riskEngine contract's isWithdrawAllowed()/isBorrowAllowed()/isAccountHealthy() functions, otherwise the calculation may be incorrect.
## Impact
Calculations in the isWithdrawAllowed()/isBorrowAllowed()/isAccountHealthy() functions of the riskEngine contract may be incorrect.
## Code Snippet
```
    function liquidate(address account) external {
        if (riskEngine.isAccountHealthy(account))
            revert Errors.AccountNotLiquidatable();
...
    function isAccountHealthy(address account) external view returns (bool) {
        return _isAccountHealthy(
            _getBalance(account),
            _getBorrows(account)
        );
    }
...
    function _getBorrows(address account) internal view returns (uint) {
        if (IAccount(account).hasNoDebt()) return 0;
        address[] memory borrows = IAccount(account).getBorrows();
        uint borrowsLen = borrows.length;
        uint totalBorrows;
        for(uint i; i < borrowsLen; ++i) {
            address LTokenAddr = registry.LTokenFor(borrows[i]);
            totalBorrows += _valueInWei(
                borrows[i],
                ILToken(LTokenAddr).getBorrowBalance(account)
            );
        }
        return totalBorrows;
    }
...
    function getBorrowBalance(address account) external view returns (uint) {
        return convertBorrowSharesToAsset(borrowsOf[account]);
    }
...
    function convertBorrowSharesToAsset(uint debt) internal view returns (uint) {
        uint256 supply = totalBorrowShares;
        return supply == 0 ? debt : debt.mulDivDown(getBorrows(), supply);
    }
...
    function getBorrows() public view returns (uint) {
        return borrows + borrows.mulWadUp(getRateFactor());
    }
...
    function updateState() public {
        if (lastUpdated == block.timestamp) return;
        uint rateFactor = getRateFactor();
        uint interestAccrued = borrows.mulWadUp(rateFactor);
        borrows += interestAccrued;
        reserves += interestAccrued.mulWadUp(reserveFactor);
        lastUpdated = block.timestamp;
    }
```
## Tool used

Manual Review

## Recommendation
Call LToken's updateState() function before calling the isWithdrawAllowed()/isBorrowAllowed()/isAccountHealthy() functions of the riskEngine contract