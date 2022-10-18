xiaoming90
# Protocol Reserve Within A LToken Vault Can Be Lent Out

## Summary

Protocol reserve, which serves as a liquidity backstop or to compensate the protocol, within a LToken vault can be lent out to the borrowers.

## Vulnerability Detail

The purpose of the protocol reserve within a LToken vault is to compensate the protocol or serve as a liquidity backstop. However, based on the current setup, it is possible for the protocol reserve within a Ltoken vault to be lent out.

The following functions within the `LToken` contract show that the protocol reserve is intentionally preserved by removing the protocol reserve from the calculation of total assets within a LToken vault. As such, whenever the Liquidity Providers (LPs) attempt to redeem their LP token, the protocol reserves will stay intact and will not be withdrawn by the LPs.

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/LToken.sol#L191

```solidity
function totalAssets() public view override returns (uint) {
    return asset.balanceOf(address(this)) + getBorrows() - getReserves();
}
```

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/LToken.sol#L195

```solidity
function getBorrows() public view returns (uint) {
    return borrows + borrows.mulWadUp(getRateFactor());
}
```

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/LToken.sol#L176

```solidity
function getReserves() public view returns (uint) {
    return reserves + borrows.mulWadUp(getRateFactor())
    .mulWadUp(reserveFactor);
}
```

However, this measure is not applied consistently across the protocol. The following `lendTo` function shows that as long as the borrower has sufficient collateral to ensure their account remains healthy, the borrower could borrow as many assets from the LToken vault as they wish.

In the worst-case scenario, the borrower can borrow all the assets from the LToken vault, including the protocol reserve.

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/LToken.sol#L128

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

## Impact

The purpose of the protocol reserve within a LToken vault is to compensate the protocol or serve as a liquidity backstop. Without the protocol reserve, the protocol will become illiquidity, and there is no fund to compensate the protocol.

## Recommendation

Consider updating the `lendTo` function to ensure that the protocol reserve is preserved and cannot be lent out. If the underlying asset of a LToken vault is less than or equal to the protocol reserve, the lending should be paused as it is more important to preserve the protocol reserve compared to lending them out.

```diff
function lendTo(address account, uint amt)
    external
    whenNotPaused
    accountManagerOnly
    returns (bool isFirstBorrow)
{
    updateState();
    isFirstBorrow = (borrowsOf[account] == 0);
    
    require

    uint borrowShares;
    require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
    totalBorrowShares += borrowShares;
    borrowsOf[account] += borrowShares;

    borrows += amt;
    asset.safeTransfer(account, amt);
    
+   require(asset.balanceOf(address(this)) >= getReserves(), "Not enough liquidity for lending") 
    
    return isFirstBorrow;
}
```

## Sentiment Team
We removed reserves completely in this [PR](https://github.com/sentimentxyz/protocol/pull/236).

## Lead Senior Watson
Confirmed fix. 
