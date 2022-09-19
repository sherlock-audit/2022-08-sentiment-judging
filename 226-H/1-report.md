devtooligan
# Disallowed collateral can be borrowed or withdrawn against - HIGH

## Summary

Disallowed collateral assets are prevented from being deposited,  However, previously deposited assets currently being held in an account can still be used as collateral to borrow, even if their allowed status is toggled off.

## Vulnerability Detail

When an account owner calls `AccountManager.borrow` a health check is conducted with the RiskEngine.  This health check (`isBorrowAllowed`) sums up the value of all assets (including disallowed collateral assets) and divides by total debt.  This number is then compared with the `balanceToBorrowThreshold`.  In addition to `borrow()`, `withdraw()` also uses this same check against `_isAccountHealthy` which could result in an account owner withdrawing all non-disallowed assets and leaving only disallowed assets to collateralize a loan.

## Impact

High - If a a collateral was disallowed because of a risk or possible vulnerability then borrowing against the asset would circumvent the act of disabling the token.  

## Code Snippet
<details>
  <summary> RiskEngine.isBorrowAllowed
</summary>

```solidity

    function isBorrowAllowed(
        address account,
        address token,
        uint amt
    )
        external
        view
        returns (bool)
    {
        uint borrowValue = _valueInWei(token, amt);
        return _isAccountHealthy(
            _getBalance(account) + borrowValue,
            _getBorrows(account) + borrowValue
        );
    }



    function _isAccountHealthy(uint accountBalance, uint accountBorrows)
        internal
        pure
        returns (bool)
    {
        return (accountBorrows == 0) ? true :
            (accountBalance.divWadDown(accountBorrows) > balanceToBorrowThreshold);
    }
```
</details>


## Tool used
Manual Review

## Recommendation
It is recommended to remove disallowed assets from the health check equation for both `isBorrowAllowed` and `isWithdrawAllowed`

<details>
  <summary> RiskEngineisBorrowAllowed()
</summary>

```diff
index 49d9c67..d44079c 100644
--- a/src/core/RiskEngine.sol
+++ b/src/core/RiskEngine.sol
@@ -80,7 +80,7 @@ contract RiskEngine is Ownable, IRiskEngine {
    function isBorrowAllowed(
        address account,
        address token,
        uint amt
    )
        external
        view
        returns (bool)
    {
        uint borrowValue = _valueInWei(token, amt);
        return _isAccountHealthy(
-            _getBalance(account) + borrowValue,
+            _getBalanceCollateralAllowed(account) + borrowValue,
             _getBorrows(account) + borrowValue
         );
     }
@@ -147,15 +147,17 @@ 
-    function _getBalance(address account) internal view returns (uint) {
+    function _getBalanceCollateralAllowed(address account) internal view returns (uint) {
         address[] memory assets = IAccount(account).getAssets();
         uint assetsLen = assets.length;
         uint totalBalance;
         for(uint i; i < assetsLen; ++i) {
-            totalBalance += _valueInWei(
-                assets[i],
-                IERC20(assets[i]).balanceOf(account)
-            );
+            if (accountManager.isCollateralAllowed(assets[i])) {
+                totalBalance += _valueInWei(
+                    assets[i],
+                    IERC20(assets[i]).balanceOf(account)
+                );
+            }
         }
         return totalBalance + account.balance;
     }
```
</details>


