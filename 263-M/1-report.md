IllIllI
# Using lower bounds for asset values may lead to incorrect pricing of risk

## Summary


## Vulnerability Detail
Multiple oracles use prices that are the minimum value of the underlying assets

## Impact
Shorting one of these assets will under-price the risk of the position

## Code Snippet
These calculations use the smallest token value as the multiplier, which means the price is a lower bound:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/oracle/src/curve/Stable2CurveOracle.sol#L43-L49

https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/oracle/src/balancer/StableBalancerLPOracle.sol#L49-L54

## Tool used

Manual Review

## Recommendation
Use the same pricing formula as is done for the Uniswap oracle:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/oracle/src/uniswap/UniV2LPOracle.sol#L42-L49