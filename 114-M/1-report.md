bin2chen
# UniV2Controller.sol miss the legal detection of tokensIn

## Summary

UniV2Controller.sol removeLiquidity() and removeLiquidityEth() do not check the legitimacy of tokensIn, resulting in the possibility of adding any token as an asset.

## Vulnerability Detail

AccountManager.sol exec() does addAsset() operation on the TokensIn returned by the called controller
But UniV2Controller's removeLiquidity() and removeLiquidityEth() are not detected the legitimacy of tokenIn,.

## Impact

possibility of adding any token as an asset.

## Code Snippet



UniV2Controller.sol 
```
    function removeLiquidity(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address tokenA, address tokenB) = abi.decode(data, (address, address));

        address[] memory tokensIn = new address[](2);
        tokensIn[0] = tokenA;
        tokensIn[1] = tokenB;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);
        return(true, tokensIn, tokensOut); //**** without check tokensIn allow? ****//
    }
```

AccountManager.sol

```
    function exec(
        address account,
        address target,
        uint amt,
        bytes calldata data
    )
        external
        onlyOwner(account)
    {
      ....
      ....
        (isAllowed, tokensIn, tokensOut) =
            controller.canCall(target, (amt > 0), data);
        if (!isAllowed) revert Errors.FunctionCallRestricted();
        _updateTokensIn(account, tokensIn);   //***** add tokensIn to asset*****//
       ....
    }
```

## Tool used

Manual Review

## Recommendation

```
    function removeLiquidity(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address tokenA, address tokenB) = abi.decode(data, (address, address));

        address[] memory tokensIn = new address[](2);
        tokensIn[0] = tokenA;
        tokensIn[1] = tokenB;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);
---     return(true, tokensIn, tokensOut);
+++     return(controller.isTokenAllowed(tokensIn[0]) && controller.isTokenAllowed(tokensIn[1]), tokensIn, tokensOut); 
    }
```

same as removeLiquidityEth()
