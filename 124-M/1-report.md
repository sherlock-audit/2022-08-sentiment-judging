xiaoming90
# Re-entrancy Risk Within The `withdraw` Function

## Summary

The withdraw function is subjected to re-entrancy risk if a token defines a hook that calls the recipient before transfer.

## Vulnerability Detail

The ERC777 contains a `_beforeTokenTransfer` hook that will be triggered before a transfer takes place. Note that the `_beforeTokenTransfer` function is triggered before the user's balance is deducted in the token contract.

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC777/ERC777.sol

```solidity
function _beforeTokenTransfer(
    address operator,
    address from,
    address to,
    uint256 amount
) internal virtual {}
```

If a token that is approved by Sentiment overwrites the `_beforeTokenTransfer` function and calls the recipient address for some valid reason, the control will be passed to the recipient before the balance is deducted. For instance, the following is an example of a vulnerable token:

```solidity
contract XYZToken is ERC777 {
	..SNIP..
    function _beforeTokenTransfer(
        address operator,
        address from,
        address to,
        uint256 amount
    ) external override {
    	to.somefunction(); // call recipient to execute some function
    }
	..SNIP..
}
```

If such a token ends up being supported by Sentiment, the `withdraw` function will be vulnerable to a re-entrancy attack and an attacker can drain all the assets within their Sentiment account.

The following POC demonstrates how an attacker could carry out the attack:

1. The attacker calls the `withdraw` function for the first time and passes the validation check (isWithdrawAllowed) at Line 187 of `AccountManager` contract
2. At Line 189, a token transfer is executed, and the `_beforeTokenTransfer` will be triggered, thus passing the control to the recipient (attacker)
3. Since the attacker is in control now, he re-enters the `withdraw` function again. Since the attacker's balance has not been deducted within the token contract yet, he will pass the validation check (isWithdrawAllowed) at Line 187 again.
4. At Line 189, the token transfer is executed again, and the `_beforeTokenTransfer` will be triggered, thus passing the control to the recipient (attacker)
5. Repeat the above steps multiple times
6. The inner calls will eventually return, and the token will be transferred to the attacker multiple times until all the assets are drained within his Sentiment account.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/core/AccountManager.sol#L183

```solidity
File: AccountManager.sol
183:     function withdraw(address account, address token, uint amt)
184:         external
185:         onlyOwner(account)
186:     {
187:         if (!riskEngine.isWithdrawAllowed(account, token, amt))
188:             revert Errors.RiskThresholdBreached();
189:         account.withdraw(msg.sender, token, amt);
190:         if (token.balanceOf(account) == 0)
191:             IAccount(account).removeAsset(token);
192:     }
```

## Impact

A malicious user can drain all their assets within their Sentiment account, thus bypassing the collateral threshold restriction. The protocol will become insolvent, and the Sentiment account is backed with no collaterals/assets.

## Recommendation

Consider implementing a re-entrancy guard on the `withdraw` function. This will eliminate the risk of a re-entrancy attack entirely even if the token implementation contains a hook that will pass the control to the recipient. The security benefits of a re-entrancy guard outweigh the gas saved.

Alternatively, a quick fix will be to move the `isWithdrawAllowed` to the end of the function.

```diff
function withdraw(address account, address token, uint amt)
    external
    onlyOwner(account)
{
-    if (!riskEngine.isWithdrawAllowed(account, token, amt))
-        revert Errors.RiskThresholdBreached();
    account.withdraw(msg.sender, token, amt);
    if (token.balanceOf(account) == 0)
        IAccount(account).removeAsset(token);
+    if (!riskEngine.isWithdrawAllowed(account, token, amt))
+        revert Errors.RiskThresholdBreached();
}
```

By doing so, even if a re-entrancy occurs, the function will verify that the account is still healthy at the end of the transaction. If the account is undercollateralized after withdrawal, the transaction will revert.

If the protocol team chooses to accept the risk by not implementing a re-entrancy guard or updating the function, the team must ensure the following are in place so that this issue does not occur:

- Before adding a new token to Sentiment, the protocol team or the auditor must ensure that the token's `_beforeTokenTransfer` do not call the recipient under any circumstance
- If a token contract added to Sentiment is upgradeable, the team must continuously monitor for any changes to the token contract and ensure that the contract does not implement a  `_beforeTokenTransfer` function that calls the recipient after the upgrade. If such an implementation is found, the team must proceed to remove the vulnerable token from Sentiment promptly

Having said that, implementing the fix would be an easier option to mitigate this risk.