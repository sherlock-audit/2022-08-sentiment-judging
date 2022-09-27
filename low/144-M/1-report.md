berndartmueller
# Balancer pool tokens with `decimals` larger than 18 are not supported

## Summary

Calculating the oracle token price of specific Balancer pool tokens with > 18 decimals will always revert.

## Vulnerability Detail

In the `WeightedBalancerLPOracle.getPrice` function, calculating the token price of a Balancer pool token with decimals higher than 18 will always revert.

## Impact

Specific Balancer pool tokens can not be appropriately used with the Sentiment protocol. Worst case, a user deposited Balancer LP tokens as collateral and cannot withdraw them due to the reverting oracle.

## Code Snippet

[oracle/src/balancer/WeightedBalancerLPOracle.sol#L60](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/balancer/WeightedBalancerLPOracle.sol#L60)

```solidity
function getPrice(address token) external view returns (uint) {
    (
        address[] memory poolTokens,
        uint256[] memory balances,
    ) = vault.getPoolTokens(IPool(token).getPoolId());

    uint256[] memory weights = IPool(token).getNormalizedWeights();

    uint length = weights.length;
    uint temp = 1e18;
    uint invariant = 1e18;
    for(uint i; i < length; i++) {
        temp = temp.mulDown(
            (oracleFacade.getPrice(poolTokens[i]).divDown(weights[i]))
            .powDown(weights[i])
        );
        invariant = invariant.mulDown(
            (balances[i] * 10 ** (18 - IERC20(poolTokens[i]).decimals())) // @audit-info Will revert for pool tokens with > 18 decimals
            .powDown(weights[i])
        );
    }
    return invariant
        .mulDown(temp)
        .divDown(IPool(token).totalSupply());
}
```

## Tools Used

Manual review

## Recommendation

Consider checking if decimals > 18 and handling them appropriately.
