minera
# Controllers: Missing isTokenAllowed check

## Summary
`CompoundController`, `ERC4626Controller`, the curve controllers, and `YearnController` do not check if the returned tokens are allowed.

## Vulnerability Detail
In `CompoundController`, `ERC4626Controller`, the curve controllers, and `YearnController`, it is not checked if the returned token is allowed. 

## Impact
Those tokens can be used as collateral (because they are added to the accounts asset afterwards) even if they are not allowed by the protocol.
Furthermore, if the token is a malicious token that reverts on `safeTransfer`, calling `sweepTo` (and therefore also liquidate) will not be possible, as it will always revert.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/controller/src/compound/CompoundController.sol#L47

## Tool used

Manual Review

## Recommendation
```
            return(controllerFacade.isTokenAllowed(tokensIn[0]), tokensIn, new address[](0));
```