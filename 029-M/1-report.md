Bahurum
# Can register a non-allowed collateral as collateral

## Summary
Some external interactions send tokens to the account, and the token address is not checked before being registered as a collateral for the account. This allows using an arbitrary token as a collateral as long as there is a corresponding oracle. 

## Vulnerability Detail
In the following controllers, at the line referenced

[UniV2Controller.sol #L189](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/uniswap/UniV2Controller.sol#L189)  
[AaveV2Controller.sol #L91](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV2Controller.sol#L91)  
[AaveV3Controller.sol #L79](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV3Controller.sol#L79)  
[BalancerController.sol#L137](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/balancer/BalancerController.sol#L137)  

There is an operation with a return value that can be an arbitrary token. No check is made to verify that the tokens in `tokensIn` are activated in `controllerFacade.isTokenAllowed`. No check is made either in `AccountManager.exec()` when calling `_updateTokensIn()` ([AccountManager.sol#L305](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L305)), so the tokens are added as a collateral as long as the oracle for the token in `OracleFacade` is active.

This can be clarified with an example using `UniV2Controller.sol`:  
1. Admin sets the oracle for token XYZ, but does not activate yet the token in `controllerFacade.isTokenAllowed` or in `AccountManager.isCollateralAllowed` waiting to clear some security concerns with the token (vulnerabilities, high volatility, low liquidity, ...)
2. Attacker opens an account
3. Attacker provides liquidity to XYZ / ETH pool on UniswapV2
4. Attacker transfers the LP to the account
5. Attacker removes liquidity from the UniswapV2 pool through the account by calling `AccountManager.exec`. Note that there is no check to control whether tokens exiting the account are allowed ([UniV2Controller.sol#L175-L190](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/uniswap/UniV2Controller.sol#L175-L190)). No checks are performed on tokens sent from the pool to the account, so account receives WETH + XYZ and also XYZ is registered as an account collateral ([AccountManager.sol#L305](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L305)). An oracle was set for XYZ so the call to `riskEngine.isAccountHealthy(account)`([AccountManager.sol#L310](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L310)) does not revert and the transaction succeeds.
6. Attacker can take advantage of the aforementioned security concerns to manipulate the price of XYZ, inflate his collateral balance and drain funds from the protocol.

The same behaviour can be reproduced with the other integrations listed above by transfering LP tokens of that particular protocol directly to an account and then removing liquidity through `AccountManager.exec()` to get the pair tokens registered into the account as collateral.

## Impact
See point 6. of section above.

## Code Snippet
[UniV2Controller.sol #L189](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/uniswap/UniV2Controller.sol#L189)  
[AaveV2Controller.sol #L91](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV2Controller.sol#L91)  
[AaveV3Controller.sol #L79](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV3Controller.sol#L79)  
[BalancerController.sol#L137](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/balancer/BalancerController.sol#L137)  
[AccountManager.sol#L347](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L347)

## Tool used

Manual Review

## Recommendation

Check if tokens are allowed as collateral in `_updateTokensIn`  ([AccountManager.sol#L347-L355](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L347-L355))

```diff
    function _updateTokensIn(address account, address[] memory tokensIn)
        internal
    {
        uint tokensInLen = tokensIn.length;
        for(uint i; i < tokensInLen; ++i) {
+           if (!isCollateralAllowed[tokensIn[i]])
+               revert Errors.CollateralTypeRestricted();
            if (IAccount(account).hasAsset(tokensIn[i]) == false)
                IAccount(account).addAsset(tokensIn[i]);
        }
    }
```

This way no asset can be added as collateral without being allowed.

## Sentiment Team
Fixed as recommended. PR [here](https://github.com/sentimentxyz/controller/pull/40) and [here](https://github.com/sentimentxyz/controller/pull/43).

## Lead Senior Watson
Confirmed fix. If there are non-whitelisted tokens in the tokensIn list from a controller, it seems like the system should not reject the call, but not to add these tokens (ignore these tokens).

This issue is more obvious for the case of claim rewards from 3rd party protocols, which may include non-whitelisted tokens, but that doesn't mean it should not be allowed to claim these rewards.

## Sentiment Team
We intend to track rewards as well and hence will be required to add them in the tokensIn list, all rewards will be whitelisted and only those integrations will be enabled.
