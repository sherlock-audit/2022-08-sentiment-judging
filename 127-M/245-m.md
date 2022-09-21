Bahurum
# Controllers do not check for receiver-like parameters

## Summary
Many of the functions of the interactions have a parameter that allows the caller to send the tokens resulting from an operation to another address. This is not checked in the controllers. While it seems that this cannot cause major losses for the protocol, it allows potentially dangerous actions.

## Vulnerability Detail
The vulnerability occurs in many integrations:
- [`AaveEthController`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveEthController.sol#L52): missing check on param `onBehalfOf` in `deposit()` and param `to` in `withdraw()`. See WETHGateway [docs](https://docs.aave.com/developers/periphery-contracts/wethgateway)
- [`AaveV2Controller`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV2Controller.sol): missing check on param `onBehalfOf` in `deposit()` and param `to` in `withdraw()`. See Aave V2 pool [docs](https://docs.aave.com/developers/v/2.0/the-core-protocol/lendingpool) 
- [`AaveV3Controller`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/aave/AaveV3Controller.sol): missing check on param `onBehalfOf` in `supply()` and param `to` in `withdraw()`. See Aave V3 pool [docs](https://docs.aave.com/developers/core-contracts/pool) 
- [`BalancerController`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/balancer/BalancerController.sol): missing check of `recipient` in the three functions
  - `swap()`: the field `recipient` of struct param [FundManagment]. See [here](https://dev.balancer.fi/resources/swaps/single-swap#fundmanagement-struct)
  - `join()`: param `recipient`. See [here](https://dev.balancer.fi/resources/joins-and-exits/pool-joins#api)
  - `exit()`: param `recipient`. See [here](https://dev.balancer.fi/resources/joins-and-exits/pool-exits#api)
  

- [`ERC4626Controller`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/erc4626/ERC4626Controller.sol): missing check for `receiver` parameter for `deposit()`, `mint()`, `redeem()` and `withdraw()`. See [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)
- [`UniV3Controller`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/uniswap/UniV3Controller.sol): missing check of `recipient` field in param structs `ExactInputSingleParams` (for `exactInputSingle()`), `ExactInputParams` (for `exactInput()`), `ExactOutputSingleParams` (for `exactOutputSingle()`), `ExactOutputParams` (for `exactOutput()`)and for `unwrapEth()`. See [docs](https://docs.uniswap.org/protocol/reference/periphery/interfaces/ISwapRouter#parameter-structs)

While the check `isAccountHealthy()` at the end of `exec()` prevents the account from being drained, having an external receiver allows reaction on sent ETH or tokens, so also reentrancy.
- transfer of ETH. This occurs in `AaveEthController` and `UniV3Controller`
- ERC777 transfer. Can occurr in any of the interactions in the list above.
- on `UniV3Controller` with `multiCall()` when one of the tokens is a malicious token.

Also, an user can use the value sent to her to do any operations as long as, before the end of the reentrant call, she is returning to the account an amount of tokens which is enough to pass the final account health check.
Example: 
- user deposits 1 ETH and borrows 4 WETH
- user withdraws 4 WETH through `AaveEthController` to her contract.
- contract executes on receive any list of arbitrary calls using the 4 ETH received.
- contract sends 4 ETH back to the account
- exec() resumes and final account health check passes.

Also, an user can in practice obtain a flash loan of an amount up to the pools size starting from very little collateral. For example:
1. user opens many accounts through an owned contract
2. contract deposits 1 ETH to the first account
3. contract borrows for the amount of 4 WETH 
4. contract withdraws 4 WETH through `AaveEthController` from the account to itself.
5. contract reacts on receive and sends 4 ETH to the second account and borrows as in point 3, but this time with the 2nd account and for an amount of 16 WETH. 
6. contract withdraws 16 WETH through `AaveEthController` from the account to itself.
7. repeat this each time sending the ETH received to another account, borrowing 4 times as much WETH and sending it back to the contract as ETH until the lending pool is empty. 
8. contract does whatever it wants with the total amount of ETH in the pool.
9. contract sends the right amount of ETH to each used account to repay its debt (4, 16, 64, ...).
10. health checks on all used accounts pass.

## Impact
Loan can be used to do arbitrary operations. Also, allows flash loans in practice. See above for details.

## Code Snippet
The occurrencies in the code of the vulnerability are linked above.

## Tool used

Manual Review

## Recommendation

Check the parameter that determines the receiver of the value-bearing tokens in an operation. Add the `address receiver` param to the `canCall()` function in the controllers and pass to it `account` in `AccountManager.exec()`([L303](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L303)). An example of receiver check is shown for `AaveEthController`:

```diff
-   function canCall(address, bool, bytes calldata data)
+   function canCall(address, bool, bytes calldata data, address receiver)
        external
        view
        returns (bool, address[] memory, address[] memory)
    {
        bytes4 sig = bytes4(data);
        if (sig == DEPOSIT) {
-           address asset = abi.decode(
+           (address asset,,address to) = abi.decode(
                data[4:],
                (address),
+               (uint256),
+               (address)
            );
            address[] memory tokensIn = new address[](1);
            address[] memory tokensOut = new address[](1);
            (tokensIn[0],,) = dataProvider.getReserveTokensAddresses(asset);
            tokensOut[0] = asset;
            return (
-               controllerFacade.isTokenAllowed(tokensIn[0]),
+               controllerFacade.isTokenAllowed(tokensIn[0]) && receiver==to,
                tokensIn,
                tokensOut
            );
        }
        if (sig == WITHDRAW) {
-           address asset = abi.decode(
+           (address asset,,address to) = abi.decode(
                data[4:],
                (address),
+               (uint256),
+               (address)
            );
            address[] memory tokensIn = new address[](1);
            address[] memory tokensOut = new address[](1);
            tokensIn[0] = asset;
            (tokensOut[0],,) = dataProvider.getReserveTokensAddresses(asset);
-           return (true, tokensIn, tokensOut);
+           return (receiver==to, tokensIn, tokensOut);
        }
        return (false, new address[](0), new address[](0));
    }
```

All other controllers impacted should be modified in a similar way.