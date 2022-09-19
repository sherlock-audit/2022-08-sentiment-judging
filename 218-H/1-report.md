__141345__
# Oracle updates could cause DoS and fail liquidation

## Summary

Oracle query could fail in some cases, causing denial of service in `isAccountHealthy()` function in `RiskEngine`. All the borrow and liquidation functions will be down. The pool involved could become insolvent due to liquidation non-functioning and assets locked in the account.

Oracle might fail for many reasons, external dependency, by mistakes, or due to malicious user's abuse. A more robust handling of oracle error is recommended to mitigate the influence of oracle mis-function.



## Vulnerability Detail

The oracle query might revert causing DoS due to the following:
- Chainlink deprecated oracle
Some assets will no longer be supported due to various reasons, those oracles will be outdated and deprecated.
- maintenance mistake
Due to the oracle availability or other reasons, some new oracles might be added, some existing ones might be removed, the collaterals allowance status are also relevant. If the oracle for some asset is not available anymore, the `OracleFacade` need to update the inner mapping `oracle[]` for record. And in contract `AccountManager`, the mapping `isCollateralAllowed[]` also need updates. Moreover, some assets in sentiment is built on other assets, such as UniSwap LP, not only the underlying individual token oracle needs update, the whole process need to be done again for the derived LP asset.
If any of these actions is missed, some assets without corresponding oracle could be included in the account assets/borrow array, the call to check account healthiness will revert due to oracle error. 
- for existing assets in assets/borrow array, the corresponding oracles are not supported anymore.

The consequence of DoS is, all the borrow and liquidation functions will be down. Without liquidation, the protocol might be insolvent.


### abuse Chainlink oracle depreciation 

If the Chainlink oracle is about to be depreciated, a malicious user can abuse it. The user can borrow the underlying asset for 1 wei, then this account will not be liquidated after the depreciation due to DoS.

Belows are the malicious user's possible abuse behaviors:

#### Avoid liquidation

Due to oracle DoS, it is impossible to check healthiness with `isAccountHealthy()`, which is the first step of `liquidate()`. 

A malicious user can do the following:
1. leverage the protocol to the limit and take a risky position.
2. borrow the asset to be depreciated for 1 wei to cause DoS. Then no need to worry about liquidation.
3. wait until the market condition in favor of the position and make a profit.
4. pay back the loans and withdraw.

Hence the user can create leveraged portfolios, no need to worry about liquidation, but still capture the up trend for the positions. The risk of losing is left for the pool.

#### Risk free leveraged portfolios

As the [docs](https://docs.sentiment.xyz/misc/useCases) suggests, sentiment can help to short one asset against another. Then an even trickier way is to create 2 opposite portfolios in 2 different accounts. For example, create 2 portfolios each with \$1,000 and leverage to \$1,500 in face value (\$500 loan), one for long ETH against USDC, another short ETH against USDC. Then one portfolio is guaranteed to profit, the other guaranteed to loss, and the profit should be the same as the loss in amount. So when the market does not fluctuate too much, the profit and loss break even. But when market has a large move in one side, the user can make a profit with no risk. Since liquidation does not work here, the profit is unlimited, the downside is limited to the initial \$1,000 collateral.

Let's say ETH price doubled, then the long ETH account gains \$1,500, net value raises to \$3,000. The other one should have lost \$1,500, but now it is a bad debt with everything locked in the account.

Result:
- the malicious user gain \$500
put down \$1,000 * 2 = \$2,000
at the end withdraw from the profit account for \$2,500 (\$3,000 face value - \$500 loan)
total gain \$500 (\$2,500 - \$1,000 collateral - \$1,000 collateral in the other account = \$500).
- pool loses \$500 loan in the beginning.


## Impact

- liquidation and borrow functions not working
- some of the loan could not be paid back
- users' collateral and fund locked in the account
- the pool could be insolvent


#### Reference

ChainLink deprecated feeds:
https://docs.chain.link/docs/deprecating-feeds/



## Code Snippet

The following oracle records need to be maintained regularly and timely.
```solidity
// oracle\src\core\OracleFacade.sol
    function setOracle(address token, IOracle _oracle) external adminOnly {
        oracle[token] = _oracle;
        emit UpdateOracle(token, address(_oracle));
    }

// protocol\src\core\AccountManager.sol
    function toggleCollateralStatus(address token) external adminOnly {
        isCollateralAllowed[token] = !isCollateralAllowed[token];
    }
```

If an asset oracle is not set, the price query will revert, or the deprecated Chainlink might also revert or return dangerous data. 
```solidity
// oracle\src\core\OracleFacade.sol
    function getPrice(address token) external view returns (uint) {
        if(address(oracle[token]) == address(0)) revert Errors.PriceUnavailable();
        return oracle[token].getPrice(token);
    }
```


## Tool used

Manual Review


## Recommendation

- for existing assets, enforce settlement if the corresponding oracles are not supported anymore
- check for 0 address in `setOracle()`:
```solidity
    require(address(_oracle) != address(0));
```
- keep updated about ChainLink deprecated feeds
also updated the `toggleTokenAllowance()` function in `ControllerFacade`. Maybe it is better to integrate `setOracle()` and `toggleTokenAllowance()` together, in case mistakenly forget to call one or another.
- use `try/catch` in `RiskEngine` if `oracle.getPrice(token)` reverts, and skip this asset. The remaining assets might be suffice to keep the account healthy. This could be a more robust approach.
