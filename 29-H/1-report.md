minera
# BalancerController: Can be misused to get non-approved tokens into the account

## Summary
In `BalancerController`s `canExit`, it is not checked if the `tokensIn` are allowed.

## Vulnerability Detail
When exiting a Balancer pool, it is possible to get tokens that are not allowed (i.e., where `controllerFacade.isTokenAllowed` would return false). There are two ways how this can happen:
1.) The pool was joined by the controller. In this case, the disallowed assets would have been in the account previously (otherwise, joining the pool is not possible). But this is possible, the user could have simply transferred those assets (via an ERC20 transfer) to his account.
2.) The user joined the pool outside of the controller (with an EOA) and transferred the pool token (via an ERC20 transfer) to his account.

## Impact
Because those tokens are added to the account's asset, it can be exploited to have tokens as collateral that are not allowed. For instance, let's say the protocol disallows SHIB because it is too volatile and therefore too risky. Using the method described above, a user could end up in a situation where SHIB is added to his assets and therefore considered in the collateral calculations. This puts the protocols assets at risk, because all tokens (that have a Balancer pool) can be added as collateral and therefore create account states that are too risky.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/controller/src/balancer/BalancerController.sol#L130

## Tool used

Manual Review

## Recommendation
One option would be to check `controllerFacade.isTokenAllowed` for all returned tokens. A better option IMO would be to change the design and drop all of this `isTokenAllowed` checks in the controllers. Instead, the `AccountManager` could check `isCollateralAllowed` for all returned tokens. Like that, the tokens would only need to be white-listed in one place and the solution would be less error-prone.