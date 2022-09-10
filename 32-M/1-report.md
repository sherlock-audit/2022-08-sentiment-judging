cergyk
# Assets without an oracle cause reversion

## Summary
As soon as an asset without an oracle is added to an account, it breaks an account.

## Vulnerability Detail
`RiskEnding._valueInWei` calls `oracle.getPrice`, which reverts for tokens without an oracle. Therefore, `isAccountHealthy` will revert as soon as an account contains an asset where no price feed exists.

## Impact
A user might get 10,000$ in USDC and 0.0001 SHIB back by an operation (e.g., by a balancer pool exit). In practice, he off course will not care about this 0.0001 SHIB and it has absolutely no impact on his health ratio. However, the execution still fails because of it.

In different controllers, is not validated properly if the returned assets are allowed (reported as a separate issue), which makes this worse.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/RiskEngine.sol#L183

## Tool used

Manual Review

## Recommendation
In my opinion, it would be better to return a price of 0 for those assets. In situations as the ones described above, the execution would succeed. And like that, those assets would still not be considered for the health ratio calculation, which is desired.