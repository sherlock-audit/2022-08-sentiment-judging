hansfriese
# `WeightedBalancerLPOracle.getPrice()` reverts when the token decimals > 18.

## Summary
`WeightedBalancerLPOracle.getPrice()` reverts when the token decimals > 18.


## Vulnerability Detail
https://github.com/sentimentxyz/oracle/tree/59b26a3d8c295208437aad36c470386c9729a4bc/src/balancer/WeightedBalancerLPOracle.sol#L60

```
invariant = invariant.mulDown(
    (balances[i] * 10 ** (18 - IERC20(poolTokens[i]).decimals()))
    .powDown(weights[i])
);
```

## Impact
`WeightedBalancerLPOracle.getPrice()` reverts when the token decimals > 18.


## Proof of Concept
As we can see [here](https://github.com/d-xo/weird-erc20#high-decimals), some tokens have more than 18 decimals.

In this case, `getPrice()` function will revert.


## Tool used
Manual Review

## Recommendation
We can modify it like below.

```
uint decimals = IERC20(poolTokens[i]).decimals();

if(decimals <= 18) {
    invariant = invariant.mulDown(
        (balances[i] * 10 ** (18 - decimals))
        .powDown(weights[i])
    );
}
else {
    invariant = invariant.divDown(
        (balances[i] * 10 ** (decimals - 18))
        .powDown(weights[i])
    );
}
```