berndartmueller
# `ERC4626Oracle` oracle calculates the wrong price for vaults with different decimals than their underlying asset

## Summary

`ERC4626` tokens with different decimals than their underlying asset will lead to overvalued account balances.

## Vulnerability Detail

It is assumed that in most cases, `ERC4626` tokens use the same amount of decimals as their underlying asset. However, it is not guaranteed and if not properly vetted before whitelisting such vault tokens as collateral, account balances are incorrectly valued and inflated.

For example, given a `ERC4626` token with 8 decimals and its underlying asset with 18 decimals. The current price calculation in `ERC4626Oracle.getPrice` will return a price with 28 decimals. However, it is expected from the oracle `getPrice` function always to return a price scaled to 18 decimals.

## Impact

Allowing a user to use `ERC4626` tokens which have different decimals than their underlying asset, will overvalue their account balance and allows users to borrow and withdraw more than anticipated.

## Code Snippet

[oracle/src/erc4626/ERC4626Oracle.sol#L35-L43](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/erc4626/ERC4626Oracle.sol#L35-L43)

```solidity
function getPrice(address token) external view returns (uint) {
  uint decimals = IERC4626(token).decimals();
  return IERC4626(token).previewRedeem(
      10 ** decimals
  ).mulDivDown(
      oracleFacade.getPrice(IERC4626(token).asset()),
      10 ** decimals
  );
}
```

## Tools Used

Manual review

## Recommendation

Consider using `decimals` of `IERC4626(token).asset()` instead of using `decimals` of the `ERC4626` vault token.
