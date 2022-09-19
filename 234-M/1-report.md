devtooligan
# `deactivate()` does not accomplish much and may lead to confusion - MEDIUM

## Summary

Calling `deactivate()` on an account does not have intended effect, 

## Vulnerability Detail

Currently,  the `Account.deactivate()` function sets the activation block to 0 but does little else. `activationBlock` is checked during the `closeAccount` function which prevents closing an account that was opened in the same block.  However, it is still possible for the accountManager to call `addAsset` or `addBorrow` on an account which has been deactivated. 

## Impact
Medium - At best this concept of activated is misleading and may lead to miscommunication or problems during future development.  At worst, the account could get into a bad state if a borrow were added to the account inadvertantly.

## Tool used
Manual Review

## Recommendation
Consider removing this concept of `inactive` altogether for increased clarity.  The name of the `activate` / `deactivate` functions could be renamed to `setActivationBlock` / `resetActivationBlock`. An alternate solution would be to add checks to `addAsset` and `addBorrow` which would prevent the Account from getting into a bad state when it was `activated`:

```diff
diff --git a/protocol/src/core/Account.sol b/protocol/src/core/Account.sol
index c17221b..ffaf307 100644
--- a/protocol/src/core/Account.sol
+++ b/protocol/src/core/Account.sol
@@ -4,7 +4,7 @@
-
 /**
     @title Sentiment Account
     @notice Contract that acts as a dynamic and distributed asset reserve
@@ -98,6 +98,7 @@ contract Account is IAccount {
         @param token Address of the ERC-20 token to add
     */
     function addAsset(address token) external accountManagerOnly {
+        require(activationBlock > 0);
         assets.push(token);
         hasAsset[token] = true;
     }
@@ -107,6 +108,7 @@ contract Account is IAccount {
         @param token Address of the ERC-20 token to add
     */
     function addBorrow(address token) external accountManagerOnly {
+        require(activationBlock > 0);
         borrows.push(token);
     }
```

