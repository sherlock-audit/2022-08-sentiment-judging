__141345__
# UniSwap V3 TWAP oracle could be manipulated

## Summary

UniSwap V3 TWAP price could potentially be artificially influenced, such as this [Rari Fuse hack](https://twitter.com/RariCapital/status/1455569653820973057). This type of oracle presents the risk of manipulation, the protocol might lose fund in some situations.



## Vulnerability Detail

For some low liquidity assets, some maybe new, those not many arbitragers are aware of. An attacker can do the following:
1. target a low liquidity UniSwap V3 pool for token A, and token B with low liquidity, few arbitrager activity.
2. deposit \$1,000 value of token A.
3. dump large amount of token B into the pool to manipulate the price.
4. wait for many blocks, let the TWAP price adjust.
5. after the TWAP price for token A inflated, the attacker will use deposited token A as collateral to the limit, for example take loan amount of \$3,000 for DAI.
6. try to take the borrowed amount out, which can be done by trade in a self controlled pool (create LP pair for DAI-USDC, then create a pool with unbalanced reserves, such as DAI-LP_DAI_USDC). Use \$1,500 DAI to swap for \$10 value LP_DAI_USDC, and remove liquidity of DAI-LP_DAI_USDC pool to take out the fund.

In this way, the healthiness in `RiskEngine` still looks good as long as the TWAP price is still high. But the borrowed amount is transferred out through unbalanced swap.

The target pool will be the low liquidity pool, also away from arbitragers.
Low liquidity will make the required amount of capital for manipulation lower, less arbitragers could help to prevent the effects of prices manipulation be negated.
Or in principal such pool could be created by the attacker. If token A and token B are legit assets, and LP for A and B is also legit can be added, because the oracle is available (UniV2LpOracle), we can call it LP_A_B. If go one step further, the next level LP can be created for LP_A_B and token A, we can call it LP_A_LA_A_B. This LP_A_LA_A_B also has a legit oracle (UniV2LpOracle), and this LP is guaranteed free from arbitrage bots.


## Impact

The protocol could lose fund due to oracle manipulation. The fund could be taken out of the account by user created unbalanced LP pair.


#### Reference

https://cmichel.io/replaying-ethereum-hacks-rari-fuse-vusd-price-manipulation/



## Code Snippet

The price feed from the UniSwap V3 TWAP oracle price would have some delay by definition:
```solidity
// oracle\src\uniswap\UniV3TWAPOracle.sol
    function getPrice(address token) public view returns (uint256) {
        // ...
        (int24 arithmeticMeanTick, ) = OracleLibrary.consult(
            pool,
            twapPeriod
        );
        // ...
    }
```

## Tool used

Manual Review

## Recommendation

Use a whitelist for assets using UniSwap V3 oracles, maybe only choose those main stream pools with enough depth.