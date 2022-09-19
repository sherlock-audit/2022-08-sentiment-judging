IllIllI
# Prices for WeightedBalancerLP tokens are not correct for pre-minted BPT pools

## Summary
Balancer docs say:
```
Note about pre-minted BPT
Some pools (like bb-a-USD) have pre-minted BPT. This means all LP tokens are minted at the time of pool creation so that you can use a swap to effectively join/exit the pool. Because of this, when querying the supply, you should NOT use bpt.getSupply(), but rather use bpt.getVirtualSupply().
```
https://dev.balancer.fi/references/lp-tokens/valuing#note-about-pre-minted-bpt

Note that while the comment says `getSupply()`, the code above the comment uses `totalSupply()`

## Vulnerability Detail
The `WeightedBalancerLPOracle` does not follow that guidance, and instead uses `totalSupply()` rather than `getVirtualSupply()`

## Impact
Mispricing of assets leading to loss of funds

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/oracle/src/balancer/WeightedBalancerLPOracle.sol#L43-L67

## Tool used

Manual Review

## Recommendation
Use `getVirtualSupply()` for pre-minted BPT
