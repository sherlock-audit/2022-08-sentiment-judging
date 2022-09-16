xiaoming90
# Liquidity Provider Unable To Withdraw Asset If High Utilisation Within A Vault

## Summary

The liquidity provider of a LToken vault will not be able to withdraw their assets if the utilization rate of the vault is high.

## Vulnerability Detail

It was observed that all the underlying assets within a LToken vault could be lent out to the borrower. There is no mechanism within the LToken vault to ensure that a certain percentage of the assets within the vault is reserved so that the liquidity provider (LP) can withdraw their asset.

The `LToken.lendTo` function below shows as long as the borrower has sufficient collateral to ensure their account remains healthy, the borrower could borrow as many assets from the LToken vault as they wish.

In the worst-case scenario, the borrower can borrow all the assets from the LToken vault, and no LP can withdraw their asset.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L128

```solidity
File: LToken.sol
121:     /**
122:         @notice Lends a specified amount of underlying asset to an account
123:         @param account Address of account
124:         @param amt Amount of token to lend
125:         @return isFirstBorrow Returns if the account is borrowing the asset for
126:             the first time
127:     */
128:     function lendTo(address account, uint amt)
129:         external
130:         whenNotPaused
131:         accountManagerOnly
132:         returns (bool isFirstBorrow)
133:     {
134:         updateState();
135:         isFirstBorrow = (borrowsOf[account] == 0);
136: 
137:         uint borrowShares;
138:         require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
139:         totalBorrowShares += borrowShares;
140:         borrowsOf[account] += borrowShares;
141: 
142:         borrows += amt;
143:         asset.safeTransfer(account, amt);
144:         return isFirstBorrow;
145:     }
```

Even if not all the assets are lent out, many of the withdrawals by LP will revert if the utilization rate of the assets within the vault is high, which means that most assets are being lent out. 

The total asset of a LToken vault is calculated based on the following formula:

```
totalAssets = asset.balanceOf(address(this)) + getBorrows() - getReserves();
```

Therefore, if most of the assets are lent out, there will not have sufficient assets for LP's withdrawal.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L191

```solidity
File: LToken.sol
185:     /**
186:         @notice Returns total amount of underlying assets
187:             totalAssets = underlying balance + totalBorrows + delta
188:             delta = totalBorrows * RateFactor
189:         @return totalAssets Total amount of underlying assets
190:     */
191:     function totalAssets() public view override returns (uint) {
192:         return asset.balanceOf(address(this)) + getBorrows() - getReserves();
193:     }
```

## Impact

LPs cannot withdraw their assets. In the worst-case scenario, assuming that a borrower has sufficient collateral to keep their account healthy, he could borrow all the assets from the LToken and keep them for a long time (e.g. two years). It might make financial sense to do so under certain economic conditions. For instance, if the token appreciation rate is higher than the borrowing rate in the long term. In this case, the users have to wait for a few years so that they can withdraw their investment if there is no new funds being injected into the LToken vault.

## Recommendation

It is recommended to adopt a more robust withdrawal mechanism for the LToken vault. Consider the following measures to mitigate the risk:

1. Enforce a debt ratio within the LToken vault to ensure that only a certain percentage (e.g. 50%) of the assets can be lent out.
2. If the LToken vault does not have sufficient underlying assets when the LPs want to withdraw their positions, consider canceling some of the outstanding positions to remove some liquidity to pay back to the LPs.

The above-recommended measures were inspired by the well-known Yearn Vault. 

Following is an extract from https://academy.moralis.io/blog/yearn-finance-what-are-yearn-vaults

> Yearn Vaults were created as a response to the yield farming craze. And their strategies continue to evolve. Also, Yearn Vaults, don’t use up all the funds on implementing a strategy. Vault holdings and strategy holdings are separate. While the Vaults put most of the funds to use in a strategy, some will sit idly in the Vault.
>
> Vaults attempt to select the idle amount first whenever a user withdrawals their funds. If the withdrawal happens to eat up more than is available in idle funds, however, then the remainder must come from the strategy’s funds. When this happens, a 0.5% fee will be assessed. Also, Ethereum gas costs must be subsidized. So, some of the profit-earning transactions will incur fees as well. 