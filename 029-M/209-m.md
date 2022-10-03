GalloDaSballo
# M-01 Account may not be able to sell a token if using UniV3

## Summary

[`UniV3Controller`](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/uniswap/UniV3Controller.sol#L134) checks to make sure that `tokenIn[0]`  is allowed.

However, no check is made for TokenOut

Meaning that if a user wants to swap into tokenX they can do it

But if they want to swap back, they may not be able to.

## Impact

This may cause situations where the user is stuck or force liquidated or forced to take a suboptimal path

## Code Snippet

## Tool used

Manual Review

## Recommendation

Change the code:
```solidity
            controllerFacade.isTokenAllowed(tokensIn[0]),
```

To
```solidity
            controllerFacade.isTokenAllowed(tokensIn[0]) &&  controllerFacade.isTokenAllowed(tokensOut[0]) 
```