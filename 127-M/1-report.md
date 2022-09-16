xiaoming90
# Malicious User Can Bypass Account Manager To Withdraw Assets

## Summary

A malicious user can bypass the account manager to withdraw their assets from the Sentiment account by interacting with external protocols.

## Vulnerability Detail

The two key characteristics of a Sentiment Account are Delegated Ownership and Controlled Interactions that Sentiment built its product upon. Refer to the [Sentiment documentation](https://docs.sentiment.xyz/core-concepts/account) for a detailed explanation of these two characteristics. Following is a brief description of the key characteristics:

- [Delegated Ownership](https://docs.sentiment.xyz/core-concepts/account#delegated-ownership) - Delegated ownership model allows the borrower to have complete control over how the assets are deployed without actually having custody of the loaned assets i.e. neither the deposited collateral nor the borrowed assets can be transferred out of the account
- [Controlled Interactions](https://docs.sentiment.xyz/core-concepts/account#controlled-interactions) - All interactions delegated by the borrower pass through the controller. The controller analyzes the calldata for the interaction to determine the type of operation and its effect on the account value

Per the design of Sentiment, if users want to withdraw any asset from the Sentiment account, they have to call the `AccountManager.withdraw` function to do so. The `AccountManager` serves as point-of-contact for the user, and all account interactions go throuthe account manager. The account manager technically serves as a gatekeeper for all the inflow and outflow of the account's assets. This approach provides various benefits as it gives Sentiment the option to implement additional validation checks or impose withdrawal fees on the incoming and outgoing assets if necessary.

However, there is another method of withdrawing the assets from the Sentiment account without going through the account manager which involves calling the Uniswap functions.

Sentiment controller supports the following Uniswap functions. Thus, the user can call these functions to interact with Uniswap.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/controller/src/uniswap/UniV2Controller.sol#L19

```solidity
File: UniV2Controller.sol
19:     /// @notice swapExactTokensForTokens(uint256,uint256,address[],address,uint256)	function signature
20:     bytes4 constant SWAP_EXACT_TOKENS_FOR_TOKENS = 0x38ed1739;
21: 
22:     /// @notice swapTokensForExactTokens(uint256,uint256,address[],address,uint256)	function signature
23:     bytes4 constant SWAP_TOKENS_FOR_EXACT_TOKENS = 0x8803dbee;
24: 
25:     /// @notice swapExactETHForTokens(uint256,address[],address,uint256) function signature
26:     bytes4 constant SWAP_EXACT_ETH_FOR_TOKENS = 0x7ff36ab5;
27: 
28:     /// @notice swapTokensForExactETH(uint256,uint256,address[],address,uint256) function signature
29:     bytes4 constant SWAP_TOKENS_FOR_EXACT_ETH = 0x4a25d94a;
30: 
31:     /// @notice swapExactTokensForETH(uint256,uint256,address[],address,uint256) function signature
32:     bytes4 constant SWAP_EXACT_TOKENS_FOR_ETH = 0x18cbafe5;
33: 
34:     /// @notice swapETHForExactTokens(uint256,address[],address,uint256) function signature
35:     bytes4 constant SWAP_ETH_FOR_EXACT_TOKENS = 0xfb3bdb41;
36: 
37:     /// @notice addLiquidity(address,address,uint256,uint256,uint256,uint256,address,uint256) function signature
38:     bytes4 constant ADD_LIQUIDITY = 0xe8e33700;
39: 
40:     /// @notice removeLiquidity(address,address,uint256,uint256,uint256,address,uint256) function signature
41:     bytes4 constant REMOVE_LIQUIDITY = 0xbaa2abde;
42: 
43:     /// @notice addLiquidityETH(address,uint256,uint256,uint256,address,uint256) function signature
44:     bytes4 constant ADD_LIQUIDITY_ETH = 0xf305d719;
45: 
46:     /// @notice removeLiquidityETH(address,uint256,uint256,uint256,address,uint256) function signature
47:     bytes4 constant REMOVE_LIQUIDITY_ETH = 0x02751cec;
```

Note that all of the above functions accept a parameter called `to` that will define the recipient of the assets. Refer to "Appendix I" section below for a detailed specification of the functions.

Therefore, an attacker could call these functions and set the `to` parameter to his own EOA address. Thus, this effectively transfers the "value" of the loaned assets from Sentiment account to his own EOA address. This will work as long as his Sentiment account remains healthy after the interaction.

The `canCall` function within the `UniV2Controller` contract only checks if the function signature is valid and if the incoming token is authorized. However, it does not verify that the recipient of the incoming tokens is the Sentiment account's address. Thus, anyone can define an arbitrary recipient via the `to` parameter of the Uniswap function.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/controller/src/uniswap/UniV2Controller.sol#L87

```solidity
File: UniV2Controller.sol
087:     function canCall(address, bool, bytes calldata data)
088:         external
089:         view
090:         returns (bool, address[] memory, address[] memory)
091:     {
092:         bytes4 sig = bytes4(data);
093: 
094:         // Swap Functions
095:         if (sig == SWAP_EXACT_TOKENS_FOR_TOKENS || sig == SWAP_TOKENS_FOR_EXACT_TOKENS)
096:             return swapErc20ForErc20(data[4:]); // ERC20 -> ERC20
097:         if (sig == SWAP_EXACT_ETH_FOR_TOKENS || sig == SWAP_ETH_FOR_EXACT_TOKENS)
098:             return swapEthForErc20(data[4:]); // ETH -> ERC20
099:         if (sig == SWAP_TOKENS_FOR_EXACT_ETH || sig == SWAP_EXACT_TOKENS_FOR_ETH)
100:             return swapErc20ForEth(data[4:]); // ERC20 -> ETH
101: 
102:         // LP Functions
103:         if (sig == ADD_LIQUIDITY) return addLiquidity(data[4:]);
104:         if (sig == REMOVE_LIQUIDITY) return removeLiquidity(data[4:]);
105:         if (sig == ADD_LIQUIDITY_ETH) return addLiquidityEth(data[4:]);
106:         if (sig == REMOVE_LIQUIDITY_ETH) return removeLiquidityEth(data[4:]);
107: 
108:         return(false, new address[](0), new address[](0));
109:     }
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/controller/src/uniswap/UniV2Controller.sol#L226

```solidity
File: UniV2Controller.sol
217:     /**
218:         @notice Evaluates whether swap can be performed
219:         @param data calldata for swapping tokens
220:         @return canCall Specifies if the interaction is accepted
221:         @return tokensIn List of tokens that the account will receive after the
222:         interactions
223:         @return tokensOut List of tokens that will be removed from the account
224:         after the interaction
225:     */
226:     function swapErc20ForErc20(bytes calldata data)
227:         internal
228:         view
229:         returns (bool, address[] memory, address[] memory)
230:     {
231:         (,, address[] memory path,,)
232:                 = abi.decode(data, (uint, uint, address[], address, uint));
233: 
234:         address[] memory tokensOut = new address[](1);
235:         tokensOut[0] = path[0];
236: 
237:         address[] memory tokensIn = new address[](1);
238:         tokensIn[0] = path[path.length - 1];
239: 
240:         return(
241:             controller.isTokenAllowed(tokensIn[0]),
242:             tokensIn,
243:             tokensOut
244:         );
245:     }
```

Assume that the attacker deposits 100 DAI into his Sentiment account, and 1 DAI == 1 USDT == 1 USDC. Uniswap's slippage and fee are ignored for simplicity's sake.

1) Attacker borrows 200 USDT from the USDT Ltoken vault
2) At this point, his total balance is $300 (100 DAI + 200 USDT) and his total borrowed is $200 (200 USDT). As such, the health ratio of his Sentiment account is around 1.5, which is above the threshold of 1.2 and the account is healthy.
3) Attacker calls `AccountManager.exec` function to perform a swap with Uniswap. He sets the `to` parameter of the swap function to his own EOA address and swaps 50 USDT (loaned) to 50 USDC.
4) Within the attacker's sentiment account, the total balance will become $250 (100 DAI + 150 USDT) and the total borrowed will remain at $200 (200 USDT). The account's health ratio will be `250/200` = 1.25, which is above the threshold of 1.2 and the account is healthy.
5) The attacker's EOA address receives 50 USDC and successfully pulls the loaned assets out from the Sentiment account without calling the `AccountManager.withdraw` function, thus effectively bypassing the account manager.

## Impact

All inflow and outflow of the account's assets should be handled via the Account Manager. If additional endpoints can be used to withdraw account's assets without going through the Account Manager, this serves as an additional attack surface, and an attacker could potentially exploit this to launch an attack.

Additionally, if Sentiment decides to enforce new validation checks or impose withdrawal fees on outgoing assets in the future within the Account Manager's withdraw function, a malicious user can workaround/bypass it by exploiting this issue.

## Recommendation

Implement additional validation to ensure that the recipient is a sentiment account so that the assets will not be transferred out of Sentiment boundary without going through the Account Manager (gatekeeper). All sentiment accounts are recorded within the Sentiment's Registry contract. Therefore, the registry can be utilized to verify if the `to` parameter points to a Sentiment account.

```diff
function swapErc20ForErc20(bytes calldata data)
    internal
    view
    returns (bool, address[] memory, address[] memory)
{
    (,, address[] memory path, address to,)
            = abi.decode(data, (uint, uint, address[], address, uint));
            
+   require(registry.ownerFor(to) != address(0), "Recipient is not a sentiment account")

    address[] memory tokensOut = new address[](1);
    tokensOut[0] = path[0];

    address[] memory tokensIn = new address[](1);
    tokensIn[0] = path[path.length - 1];

    return(
        controller.isTokenAllowed(tokensIn[0]),
        tokensIn,
        tokensOut
    );
}
```

## Appendix I

Following are the specification of the supported functions taken from https://docs.uniswap.org/protocol/V2/reference/smart-contracts/router-02

Note: The `to` parameter defines the recipient of the assets.

swapExactTokensForTokens

```solidity
function swapExactTokensForTokens(
  uint amountIn,
  uint amountOutMin,
  address[] calldata path,
  address to,
  uint deadline
) external returns (uint[] memory amounts);
```

swapTokensForExactTokens

```solidity
function swapTokensForExactTokens(
  uint amountOut,
  uint amountInMax,
  address[] calldata path,
  address to,
  uint deadline
) external returns (uint[] memory amounts);
```

swapExactETHForTokens

```solidity
function swapExactETHForTokens(uint amountOutMin, address[] calldata path, address to, uint deadline)
  external
  payable
  returns (uint[] memory amounts);
```

swapTokensForExactETH

```solidity
function swapTokensForExactETH(uint amountOut, uint amountInMax, address[] calldata path, address to, uint deadline)
  external
  returns (uint[] memory amounts);
```

swapExactTokensForETH

```solidity
function swapExactTokensForETH(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline)
  external
  returns (uint[] memory amounts);
```

swapETHForExactTokens

```solidity
function swapETHForExactTokens(uint amountOut, address[] calldata path, address to, uint deadline)
  external
  payable
  returns (uint[] memory amounts);
```

addLiquidity

```solidity
function addLiquidity(
  address tokenA,
  address tokenB,
  uint amountADesired,
  uint amountBDesired,
  uint amountAMin,
  uint amountBMin,
  address to,
  uint deadline
) external returns (uint amountA, uint amountB, uint liquidity);
```

removeLiquidity

```solidity
function removeLiquidity(
  address tokenA,
  address tokenB,
  uint liquidity,
  uint amountAMin,
  uint amountBMin,
  address to,
  uint deadline
) external returns (uint amountA, uint amountB);
```

addLiquidityETH

```solidity
function addLiquidityETH(
  address token,
  uint amountTokenDesired,
  uint amountTokenMin,
  uint amountETHMin,
  address to,
  uint deadline
) external payable returns (uint amountToken, uint amountETH, uint liquidity);
```

removeLiquidityETH

```solidity
function removeLiquidityETH(
  address token,
  uint liquidity,
  uint amountTokenMin,
  uint amountETHMin,
  address to,
  uint deadline
) external returns (uint amountToken, uint amountETH);
```