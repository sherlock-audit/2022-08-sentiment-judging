xiaoming90
# Token Without Price Oracle Can Cause Asset To Be Locked

## Summary

If a token within the Sentiment account does not have a price oracle, it will cause the assets within the account to be locked.

## Vulnerability Detail

During depositing, the protocol will verify that the deposited token is allowed by calling the `isCollateralAllowed` function at Line 167 to ensure that tokens not approved to be used within the protocol cannot be deposited to a Sentiment account.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/core/AccountManager.sol#L162

```solidity
File: AccountManager.sol
162:     function deposit(address account, address token, uint amt)
163:         external
164:         whenNotPaused
165:         onlyOwner(account)
166:     {
167:         if (!isCollateralAllowed[token])
168:             revert Errors.CollateralTypeRestricted();
169:         if (IAccount(account).hasAsset(token) == false)
170:             IAccount(account).addAsset(token);
171:         token.safeTransferFrom(msg.sender, account, amt);
172:     }
```

However, this validation is not consistently implemented across the protocol. The protocol does not validate the input tokens received from the account interaction with external protocols.

Assume that Sentiment accepts the [Balancer's 50OHM-25DAI-25WETH LP token](https://app.balancer.fi/#/pool/0xc45d42f801105e861e86658648e3678ad7aa70f900010000000000000000011e) as collateral and Alice deposits some 50OHM-25DAI-25WETH LP tokens into her Sentiment account as collateral. At some point later, Alice wants to sell all her 50OHM-25DAI-25WETH LP tokens in her Sentiment account. Therefore, she calls the `AccountManager.exec` function with the ["exit" option](https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/controller/src/balancer/BalancerController.sol#L21) to instruct Sentiment to exit the pool on her behalf. The Sentiment Controller will then trigger the balancer's `exitPool` function and transfer all her 50OHM-25DAI-25WETH LP tokens to Balancer. In return, the caller will receive underlying assets of the weighted pools (OHM, DAI, WETH).

At Line 305 of the `AccountManager` contract, it will call the `_updateTokensIn` function to add the received tokens (OHM, DAI, WETH) into the asset list of Alice's Sentiment account.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/core/AccountManager.sol#L290

```solidity
File: AccountManager.sol
290:     function exec(
291:         address account,
292:         address target,
293:         uint amt,
294:         bytes calldata data
295:     )
296:         external
297:         onlyOwner(account)
298:     {
299:         bool isAllowed;
300:         address[] memory tokensIn;
301:         address[] memory tokensOut;
302:         (isAllowed, tokensIn, tokensOut) =
303:             controller.canCall(target, (amt > 0), data);
304:         if (!isAllowed) revert Errors.FunctionCallRestricted();
305:         _updateTokensIn(account, tokensIn);
306:         (bool success,) = IAccount(account).exec(target, amt, data);
307:         if (!success)
308:             revert Errors.AccountInteractionFailure(account, target, amt, data);
309:         _updateTokensOut(account, tokensOut);
310:         if (!riskEngine.isAccountHealthy(account))
311:             revert Errors.RiskThresholdBreached();
312:     }
```

Observed that within the `_updateTokensIn` function, it does not check whether the input token is supported as collateral within the Sentiment and simply adds the input token into the Sentiment account's asset list.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/core/AccountManager.sol#L347

```solidity
File: AccountManager.sol
347:     function _updateTokensIn(address account, address[] memory tokensIn)
348:         internal
349:     {
350:         uint tokensInLen = tokensIn.length;
351:         for(uint i; i < tokensInLen; ++i) {
352:             if (IAccount(account).hasAsset(tokensIn[i]) == false)
353:                 IAccount(account).addAsset(tokensIn[i]);
354:         }
355:     }
```

After executing the transaction, the asset list of Alice's Sentiment account contains the following tokens:

```solidity
OHM, DAI, WETH
```

Assume that the following:

- DAI and WETH are supported by Sentiment and have price oracle configured
- OHM is not supported by Sentiment, and thus there is no price oracle configured

Whenever the `_getBalance` function is called on Alice's Sentiment account, the function will revert.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/core/RiskEngine.sol#L150

```solidity
File: RiskEngine.sol
150:     function _getBalance(address account) internal view returns (uint) {
151:         address[] memory assets = IAccount(account).getAssets();
152:         uint assetsLen = assets.length;
153:         uint totalBalance;
154:         for(uint i; i < assetsLen; ++i) {
155:             totalBalance += _valueInWei(
156:                 assets[i],
157:                 IERC20(assets[i]).balanceOf(account)
158:             );
159:         }
160:         return totalBalance + account.balance;
161:     }
```

This is because the`getBalance` function will attempt to loop through all the tokens (OHM, DAI, WETH) within Alice's Sentiment account and calls the `oracle.getPrice(token)` for each of them. Since there is no price oracle within Sentiment for OHM, the function will revert.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/core/RiskEngine.sol#L178

```solidity
File: RiskEngine.sol
178:     function _valueInWei(address token, uint amt)
179:         internal
180:         view
181:         returns (uint)
182:     {
183:         return oracle.getPrice(token)
184:         .mulDivDown(
185:             amt,
186:             10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
187:         );
188:     }
```

Since `isBorrowAllowed`, `isWithdrawAllowed`, and `isAccountHealthy` functions rely on `_getBalance` function, many of the critical functions within Sentiment will no longer work.

## Impact

Following are some impacts caused by this issue:

- Since `isBorrowAllowed` reverts, users can no longer borrow any asset
- Since `isWithdrawAllowed` reverts, users can no longer withdraw their assets within the Sentiment account. This effectively causes the user fund to be locked.
- Since `isAccountHealthy` reverts, an unhealthy Sentiment account cannot be liquidated
- Since `isAccountHealthy` reverts, users cannot call `AccountManager.exec` to perform any interaction with an external protocol.

## Recommendation

It is recommended to verify that an input token is supported collateral within Sentiment and that a price oracle has been configured for the input token before adding it to the user's Sentiment account.

Consider implementing additional validations within the `_updateTokensIn` function. This whitelisting approach will prevent all edge cases where certain tokens are accidentally accepted by Sentiment.

```diff
function _updateTokensIn(address account, address[] memory tokensIn)
    internal
{
    uint tokensInLen = tokensIn.length;
    for(uint i; i < tokensInLen; ++i) {
+    	if (!isCollateralAllowed[tokensIn[i]])
+    		 revert Errors.CollateralTypeRestricted();
+   	if(address(oracle.oracle[tokensIn[i]]) == address(0)) 
+   		revert Errors.PriceUnavailable();
        if (IAccount(account).hasAsset(tokensIn[i]) == false)
            IAccount(account).addAsset(tokensIn[i]);
    }
}
```