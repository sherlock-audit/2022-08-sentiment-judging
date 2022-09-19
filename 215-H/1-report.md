__141345__
# Unbounded `assets` array causing DoS to avoid liquidation and abuse the system

## Summary

Users can add arbitrary number of assets in the account, causing denial of service in `isAccountHealthy()` function in `RiskEngine`, and avoid being liquidated. All the borrow and liquidation functions will be down. A malicious user can abuse this to create risk-free leveraged portfolios to capture the market trend, leaving all the risk to the pool. The pool involved could become insolvent due to liquidation non-functioning and assets locked in the account.


## Vulnerability Detail

#### Avoid liquidation

As `assets` array in `Account` contract can grow quite large, the transaction’s gas cost of loop through the `assets` could exceed the block gas limit and makes it impossible to check healthiness with `isAccountHealthy()`, which is the first step of `liquidate()`. 

A malicious user can do the following:
1. leverage the protocol to the limit and take a risky position.
2. inflate the `assets` array to be extremely long so as to cause DoS. Then no need to worry about liquidation.
3. wait until the market condition in favor of the position and make a profit.
4. pay back the loans and withdraw.

Hence the user can create leveraged portfolios, no need to worry about liquidation, but still capture the up trend for the positions. The risk of losing is left for the pool.

#### Risk free leveraged portfolios

As the [docs](https://docs.sentiment.xyz/misc/useCases) suggests, sentiment can help to short one asset against another. Then an even trickier way is to create 2 opposite portfolios in 2 different accounts. For example, create 2 portfolios each with \$1,000 and leverage to \$1,500 in face value (\$500 loan), one for long ETH against USDC, another short ETH against USDC. Then one portfolio is guaranteed to profit, the other guaranteed to loss, and the profit should be the same as the loss in amount. So when the market does not fluctuate too much, the profit and loss break even. But when market has a large move in one side, the user can make a profit with no risk. Since liquidation does not work here, the profit is unlimited, the downside is limited to the initial \$1,000 collateral.

Let's say ETH price doubled, then the long ETH account gains \$1,500, net value raises to \$3,000. The other one should have lost \$1,500, but now it is a bad debt with everything locked in the account.

Result:
- the malicious user gain \$500:
put down \$1,000 * 2 = \$2,000
at the end withdraw from the profit account for \$2,500 (\$3,000 face value - \$500 loan)
total gain \$500 (\$2,500 - \$1,000 collateral - \$1,000 collateral in the other account = \$500).
- pool loses \$500 loan in the beginning.


## Impact

- liquidation and borrow functions not working
- some of the loan could not be paid back
- users' collateral and fund locked in the account
- the pool could be insolvent


## Code Snippet

The malicious user can call `exec()` with a long list of assets, but with each one only 1 wei in amount. Then `_updateTokensIn()` will add the token into `assets` array.

```solidity
// protocol\src\core\AccountManager.sol
    function exec(
        address account,
        address target,
        uint amt,
        bytes calldata data
    )
        external
        onlyOwner(account)
    {
        address[] memory tokensIn;

        (isAllowed, tokensIn, tokensOut) =
            controller.canCall(target, (amt > 0), data);

        _updateTokensIn(account, tokensIn);
        // ...
    }

    function _updateTokensIn(address account, address[] memory tokensIn)
        internal
    {
        uint tokensInLen = tokensIn.length;
        for(uint i; i < tokensInLen; ++i) {
            if (IAccount(account).hasAsset(tokensIn[i]) == false)
                IAccount(account).addAsset(tokensIn[i]);
        }
    }
```

The `liquidate()` function need to call `riskEngine.isAccountHealthy()` first. 
```solidity
// protocol\src\core\AccountManager.sol
    function liquidate(address account) external {
        if (riskEngine.isAccountHealthy(account))
            revert Errors.AccountNotLiquidatable();
        _liquidate(account);
    }
```

Then in `_getBalance()` the whole list of `assets` array will be looped to calculate the healthiness.
```solidity
// protocol\src\core\RiskEngine.sol
    function isAccountHealthy(address account) external view returns (bool) {
        return _isAccountHealthy(
            _getBalance(account),
            _getBorrows(account)
        );
    }

    function _getBalance(address account) internal view returns (uint) {
        address[] memory assets = IAccount(account).getAssets();
        uint assetsLen = assets.length;
        uint totalBalance;
        for(uint i; i < assetsLen; ++i) {
            totalBalance += _valueInWei(
                assets[i],
                IERC20(assets[i]).balanceOf(account)
            );
        }
        return totalBalance + account.balance;
    }
```

If the `assets` array in `Account` grows too large, the for loop will run out of gas and result in denial of service.


## Tool used

Manual Review


## Recommendation

- Limit the total number of assets in one account.
- Set minimum amount or value in each execution.
- Provide alternative way to calculate account healthiness with several parts.
