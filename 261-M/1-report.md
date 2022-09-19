IllIllI
# WeightedBalancerLPOracle is broken for tokens with more than 18 decimals

## Summary
The WeightedBalancerLPOracle assumes all tokens have fewer than 18 decimals, but there are tokens that have more

## Vulnerability Detail
The oracle does a subtraction which may underflow if the number of decimals of the underlying token is greater than 18

## Impact
Asset will not be able to be priced

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/oracle/src/balancer/WeightedBalancerLPOracle.sol#L60

## Tool used

Manual Review

## Recommendation
Use an if-else and change the unit conversion if the number of decimals is greater than 18