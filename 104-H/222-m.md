cccz
# RiskEngine: Unbounded loops may cause _getBalance()/_getBorrows() to fail

## Summary
RiskEngine: Unbounded loops may cause _getBalance()/_getBorrows() to fail
## Vulnerability Detail
Loops that do not have a fixed number of iterations, for example, loops that depend on storage values, have to be used carefully: Due to the block gas limit, transactions can only consume a certain amount of gas. Either explicitly or just due to normal operation, the number of iterations in a loop can grow beyond the block gas limit which can cause the complete contract to be stalled at a certain point.
In the _getBalance()/_getBorrows() function of the RiskEngine contract, assets/borrows are iterated. And the user can increase the length of assets/borrows when calling the deposit()/borrow() functions of the AccountManager contract.
If _getBalance()/_getBorrows() fails due to exceeding the block gas limit, then the isWithdrawAllowed()/isWithdrawAllowed()/isAccountHealthy() functions of the RiskEngine contract will also fail, thus preventing the user from from borrowing, withdrawing, or even being liquidated (the user has the incentive to prevent himself from being liquidated). 
This should be a medium vulnerability considering that the contract will limit the collateralizable and borrowable assets, but if the contract integrates many assets, this could be a high vulnerability.
## Impact
If _getBalance()/_getBorrows() fails due to exceeding the block gas limit, then the isWithdrawAllowed()/isWithdrawAllowed()/isAccountHealthy() functions of the RiskEngine contract will also fail, thus preventing the user from from borrowing, withdrawing, or even being liquidated (the user has the incentive to prevent himself from being liquidated).
## Code Snippet
```
    function _getBalance(address account) internal view returns (uint) {
        address[] memory assets = IAccount(account).getAssets();
        uint assetsLen = assets.length;
        uint totalBalance;
        for(uint i; i < assetsLen; ++i) {
            totalBalance += _valueInWei(
                assets[i],
                IERC20(assets[i]).balanceOf(account)
            );
        }
        return totalBalance + account.balance;
    }

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
```
## Tool used

Manual Review

## Recommendation

Consider limiting the length of borrows/assets in the Account contract