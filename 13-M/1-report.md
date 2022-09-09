minera
# StableBalancerLPOracle: Wrong calculation

## Summary
`StableBalancerLPOracle` calculates the rate relative to the min price of the assets, although it is calculated with respect to all assets.

## Vulnerability Detail
The rate that is returned by `getRate()` is multipled by the lowest price of all tokens in the pool. However, if we look at how the rate is actually determined (https://github.com/balancer-labs/balancer-v2-monorepo/blob/master/pkg/pool-stable/contracts/ComposableStablePool.sol#L932), we can see that it returns the rate with respect to the balance of all tokens in the pool. Therefore, this approach results in wrong values.

## Impact
In an extreme scenario, a stable coin pool has 3 tokens with prices $0.95, $1, $1. The balances are 1, 1000, 1000 and the returned rate is 1.2.
The calculation will result in $0.95 * 1.2 = $1.14, whereas the true price would be very close to $1.2
Because of this, these tokens are undervalued, which can result in unnecessary liquidations (i.e., a loss of funds).

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/balancer/StableBalancerLPOracle.sol#L54

## Tool used

Manual Review

## Recommendation
Calculate the correct price, incorporating the balance of the different tokens (see the balancer repo for details).