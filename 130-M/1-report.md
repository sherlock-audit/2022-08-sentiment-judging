xiaoming90
# Residual Borrowed Shares Cause User's Collateral To Be Stuck And Certain Functions To Be Broken

## Summary

Repayment can cause a small amount of borrowed shares to remain in user's account (`borrowsOf[user account]`) within the LToken vault. The residual borrowed shares cannot be cleared from the user's account, causing user's collateral to be stuck and certain functions (e.g. closing account, liquidation) to be broken.

## Vulnerability Detail

Assume the following state within the DAI Ltoken vault:

- `totalBorrowShares` (also known as `supply`)(total borrow shares minted) = 100 shares

- `borrows` (total amount of borrows) = 50 DAI

- `borrowsOf[Alice's account address]` = 5 shares

Alice borrowed 5 shares. Alice attempts to repay all her outstanding debt by calling the following function:

```solidity
AccountManager.repay(Alice's account address, DAI.address, type(uint256).max)
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L227

```solidity
File: AccountManager.sol
227:     function repay(address account, address token, uint amt)
228:         public
229:         onlyOwner(account)
230:     {
231:         ILToken LToken = ILToken(registry.LTokenFor(token));
232:         if (address(LToken) == address(0))
233:             revert Errors.LTokenUnavailable();
234:         LToken.updateState();
235:         if (amt == type(uint256).max) amt = LToken.getBorrowBalance(account);
236:         account.withdraw(address(LToken), token, amt);
237:         if (LToken.collectFrom(account, amt))
238:             IAccount(account).removeBorrow(token);
239:         if (IERC20(token).balanceOf(account) == 0)
240:             IAccount(account).removeAsset(token);
241:         emit Repay(account, msg.sender, token, amt);
242:     }
```

Since `amt` parameter is equal to `type(uint256).max`, the `LToken.getBorrowBalance(account)` function at Line 235 of the `AccountManager.sol` will be triggered. Within the `LToken.getBorrowBalance(account)` function, it will in turn call the `convertBorrowSharesToAsset` function.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L172

```solidity
File: LToken.sol
167:     /**
168:         @notice Returns Borrow balance of given account
169:         @param account Address of account
170:         @return borrowBalance Amount of underlying tokens borrowed
171:     */
172:     function getBorrowBalance(address account) external view returns (uint) {
173:         return convertBorrowSharesToAsset(borrowsOf[account]);
174:     }
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L234

```solidity
File: LToken.sol
234:     function convertBorrowSharesToAsset(uint debt) internal view returns (uint) {
235:         uint256 supply = totalBorrowShares;
236:         return supply == 0 ? debt : debt.mulDivDown(getBorrows(), supply);
237:     }
```

The parameter values passed to the `convertBorrowSharesToAsset` function will be as follows:

```
convertBorrowSharesToAsset(5)
```

The `convertBorrowSharesToAsset` function will execute the following code:

```solidity
debt.mulDivDown(getBorrows(), supply);
```

which is equal to

```solidity
5.mulDivDown(50, 100);
```

which is equal to

```solidity
(5 * 50) / 100 = 2 // precision loss due to how solidity handles division. 2.5 round down to 2
```

Note that with `5` shares that Alice borrowed, the borrow balance is supposed to be `2.5` DAI. However, due to how solidity handles division, there will be a precision loss. As a result, `LToken.getBorrowBalance(account)` will return only `2` DAI, and the `amt` variable will be set to `2` DAI.

Next, the `LToken.collectFrom(account, amt)` function at Line 237 of the `AccountManager.sol` will be triggered. 

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L227

```solidity
File: AccountManager.sol
227:     function repay(address account, address token, uint amt)
228:         public
229:         onlyOwner(account)
230:     {
231:         ILToken LToken = ILToken(registry.LTokenFor(token));
232:         if (address(LToken) == address(0))
233:             revert Errors.LTokenUnavailable();
234:         LToken.updateState();
235:         if (amt == type(uint256).max) amt = LToken.getBorrowBalance(account);
236:         account.withdraw(address(LToken), token, amt);
237:         if (LToken.collectFrom(account, amt))
238:             IAccount(account).removeBorrow(token);
239:         if (IERC20(token).balanceOf(account) == 0)
240:             IAccount(account).removeAsset(token);
241:         emit Repay(account, msg.sender, token, amt);
242:     }
```

The parameter values passed to the `collectFrom` function will be as follows:

```solidity
LToken.collectFrom(Alice's account address, 2)
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L153

```solidity
File: LToken.sol
147:     /**
148:         @notice Collects a specified amount of underlying asset from an account
149:         @param account Address of account
150:         @param amt Amount of token to collect
151:         @return bool Returns true if account has no debt
152:     */
153:     function collectFrom(address account, uint amt)
154:         external
155:         accountManagerOnly
156:         returns (bool)
157:     {
158:         uint borrowShares;
159:         require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
160:         borrowsOf[account] -= borrowShares;
161:         totalBorrowShares -= borrowShares;
162: 
163:         borrows -= amt;
164:         return (borrowsOf[account] == 0);
165:     }
```

Subsequently, the `convertAssetToBorrowShares(amt)` function at Line 159 of the `LToken.sol` will be triggered.

The parameter values passed to the `convertAssetToBorrowShares` function will be as follows:

```solidity
convertAssetToBorrowShares(2)
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L229

```solidity
File: LToken.sol
229:     function convertAssetToBorrowShares(uint amt) internal view returns (uint) {
230:         uint256 supply = totalBorrowShares;
231:         return supply == 0 ? amt : amt.mulDivUp(supply, getBorrows());
232:     }
```

The `convertAssetToBorrowShares` function will execute the following code:

```solidity
amt.mulDivUp(supply, getBorrows())
```

which is equal to

```solidity
2.mulDivUp(100, 50)
```

which is equal to

```solidity
(2 * 100) / 50 = 4
```

The `convertAssetToBorrowShares` function will return with `4` shares.

Line 160 and 161 of the `LToken.sol` will deduct `4` shares from `borrowsOf[account]` and `totalBorrowShares` state variables respectively.

Line 163 of the `LToken.sol` will deduct `2` DAI from `borrows` state variable.

At this point, the state within the DAI Ltoken vault is as follows:

- `totalBorrowShares` (also known as `supply`)(total borrow shares minted) = (100 - 4) = 96 shares

- `borrows` (total amount of borrows) = (50 - 2) = 48 DAI

- `borrowsOf[Alice's account address]` = (5 - 4) = 1 share

The `collectFrom` function will return a `false` boolean because `borrowsOf[Alice's account address]` is not equal to zero.

Since the `collectFrom` function returns `false`, then the `IAccount(account).removeBorrow(token);` code at Line 238 of the `AccountManager.sol` will not be executed. Thus, the `DAI` token will not be removed from the account's borrows list. 

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L237

```solidity
File: AccountManager.sol
237:         if (LToken.collectFrom(account, amt))
238:             IAccount(account).removeBorrow(token);
```

Observed that Alice has attempted to clear all her outstanding debt by calling `AccountManager.repay(Alice's account address, DAI.address, type(uint256).max)` earlier. However, the protocol fails to clear all her outstanding debt as Alice still has 1 share of the borrowed debt left. At this point, `borrowsOf[Alice's account address]` is `1` so Alice's outstanding debt is not fully cleared. This raises an issue that the clear all outstanding option within the `AccountManager.repay` is not working as intended.

More importantly, notice that the `DAI` token is NOT removed from the account's borrows list since Alice's outstanding debt is not considered as fully cleared even though she has instructed the protocol to clear all her outstanding debt by specifying `type(uint256).max)` as the amount. This will eventually cause an issue later on as the account's borrows list array will grow silently till the point where an Out-of-Gas error will occur when some function attempts to loop through the account's borrow list array. At some point later, some of the functions within the protocol will be unusable for the user when this issue happens.  How fast it happens depends on the user's trading pattern.

In addition, affected users cannot fully pay off their borrowed debt and cannot close their accounts. As such, their collaterals or funds will be stuck in the account. 

Refer to the "Impact" section for more examples.

The following two additional pieces of evidence show that it is not possible for Alice to clear all her outstanding debts even if she tried to call the functions with a different configuration.

#### Additional Note #1 - User Attempts To Clear All Debt Again By Calling `repay` with `amt=type(uint256).max`

Assume that Alice thinks that she could clear all her outstanding debt by calling the following function for the second time.

```solidity
AccountManager.repay(Alice's account address, DAI.address, type(uint256).max)
```

At this point, the state within the DAI Ltoken vault is as follows:

- `totalBorrowShares` (also known as `supply`)(total borrow shares minted) = 96 shares

- `borrows` (total amount of borrows) = 48 DAI

- `borrowsOf[Alice's account address]` = 1 share

Going through the same steps as the previous section, the following function will be triggered

```solidity
convertBorrowSharesToAsset(1)
```

which is equal to

```solidity
(1 * 48) / 96 = 0 // precision loss due to how solidity handles division. 0.5 round down to 0
```

As a result, `LToken.getBorrowBalance(account)` will return only `0` DAI, and the `amt` variable will be set to `0` DAI.

Next, the `Ltoken.collectFrom` function will be executed with the following parameter values:

```solidity
LToken.collectFrom(Alice's account address, 0)
```

Eventually, it will reach `require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");` and revert. 

Thus, the token is still not being removed from the account's borrow list array, and Alice's outstanding debt is still not cleared.

#### Additional Note #2 - User Attempts To Clear All Debt Again By Calling `repay` by manually specifying the remaining debt

Assume that Alice decides to try an alternative method by manually specifying the remaining debt when calling the `repay` function, and hoping that it will clear all her outstanding debt within the protocol. Alice calls the following:

```
AccountManager.repay(Alice's account address, DAI.address, 1)
```

At this point, the state within the DAI Ltoken vault is as follows:

- `totalBorrowShares` (also known as `supply`)(total borrow shares minted) = 96 shares

- `borrows` (total amount of borrows) = 48 DAI

- `borrowsOf[Alice's account address]` = 1 share

Eventually, the following function will be triggered:

```solidity
LToken.collectFrom(Alice's account address, 1)
```

Within the `LToken.collectFrom` function, it will call the following function:

```solidity
convertAssetToBorrowShares(1)
```

which is equal to

```solidity
(1 * 96) / 48 = 2 shares
```

As a result,  the `borrowShares` within the `LToken.collectFrom` function will be set to `2`. Observed that this value is actually wrong and will cause an issue later on as Alice is supposed to only have `1` share of debt left.

At Line 160 of LToken.sol, `borrowsOf[Alice's account address]` is equal to `1` and `borrowShares` is equal to `2`. Thus, `1 - 2` will cause an underflow and the function will revert.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L153

```solidity
File: LToken.sol
153:     function collectFrom(address account, uint amt)
154:         external
155:         accountManagerOnly
156:         returns (bool)
157:     {
158:         uint borrowShares;
159:         require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
160:         borrowsOf[account] -= borrowShares;
161:         totalBorrowShares -= borrowShares;
162: 
163:         borrows -= amt;
164:         return (borrowsOf[account] == 0);
165:     }
```

Again, the token is still not being removed from the account's borrow list array, and Alice's outstanding debt is still not cleared.

## Impact

Following is a non-exhaustive list of the functions affected by this issue:

- User cannot fully pay off their borrowed debt and cannot close their accounts. As such, user's collaterals or funds are stuck in the account. `AccountManager.closeAccount` function will call the `account.hasNoDebt()` function, and the `account.hasNoDebt()` function will only return `true` if the account's borrow list array is empty. 
- An account cannot be settled or liquidated. `AccountManager.settle` and `AccountManager._liquidate` functions will loop through the account's borrow list array, and an Out-of-Gas error will cause a revert.
- Any functions (e.g. borrow, withdraw) that triggered `RiskEngine.isBorrowAllowed` and `RiskEngine.isWithdrawAllowed` will revert as these two functions will loop through the account's borrow list array, and an Out-of-Gas error will cause a revert. 

## Recommendation

If a user instructed the protocol to clear all their outstanding debt by calling the `repay` function with `amt` set to `type(uint256).max`

```solidity
AccountManager.repay(Alice's account address, DAI.address, type(uint256).max)
```

Assuming that the user has sufficient funds to repay all their debt, at the end of the transaction:

- All the user's outstanding debt should be cleared, and there should not be any residual debt/share left in their account.
- User's `borrowsOf[account]` must be zero
- The repaid token should be cleared from the user account's borrow list array

Consider implementing the following to mitigate the issue:

```diff
function repay(address account, address token, uint amt)
    public
    onlyOwner(account)
{
    ILToken LToken = ILToken(registry.LTokenFor(token));
    if (address(LToken) == address(0))
        revert Errors.LTokenUnavailable();
    LToken.updateState();
+   bool isMax = amt == type(uint256).max;
+   if (isMax) amt = LToken.getBorrowBalance(account);
-   if (amt == type(uint256).max) amt = LToken.getBorrowBalance(account);
    account.withdraw(address(LToken), token, amt);
+    if (LToken.collectFrom(account, amt, isMax))
-    if (LToken.collectFrom(account, amt))
        IAccount(account).removeBorrow(token);
    if (IERC20(token).balanceOf(account) == 0)
        IAccount(account).removeAsset(token);
    emit Repay(account, msg.sender, token, amt);
}
    
+function collectFrom(address account, uint amt, bool isMax)
-function collectFrom(address account, uint amt)
    external
    accountManagerOnly
    returns (bool)
{
    uint borrowShares;
+   if (isMax) {
+   	borrowShares = borrowsOf[account];
+   } else 
    	require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
+   }
    borrowsOf[account] -= borrowShares;
    totalBorrowShares -= borrowShares;

    borrows -= amt;
    return (borrowsOf[account] == 0);
}
```

The above implementation ensures that all the borrowed shares will be cleared from User's `borrowsOf[account]` if they decide to clear all her outstanding debt by specifying `amt` to `type(uint256).max`. This will prevent any issues that arise due to the residual shares.