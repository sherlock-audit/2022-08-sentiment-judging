Bahurum
# ERC4626 underlying decimals could be different

## Summary
In `ERC4626Oracle.getPrice()` the number of `decimals` of the `ERC4626` token and of the underlying `asset` are assumed to be the same. This is not always true and is an implicit assumption which can lead to unknowingly integrate a `ERC4626` token that does not respect this as a collateral. This will cause gross miscalculation of `asset` price and underestimation or overestimation of an account's collateral.

## Vulnerability Detail
While the `ERC4626` standards suggests that the underlying `asset` and the token itself have the same number of `decimals`, this is not mandatory (see https://eips.ethereum.org/EIPS/eip-4626). While the most popular implementation does respect this, there are some implementations currently in use that do not. For example [AaveV2StablecoinCellar](https://etherscan.io/address/0x7bAD5DF5E11151Dc5Ee1a648800057C5c934c0d5#code) has 18 `decimals` but its underlying `asset` is `USDT` (6 `decimals`). In the same way, implementations with decimals of `asset` > decimals of `ERC4626` can exist. The `ERC4626` collateral price calculation in [`ERC4626Oracle.getPrice()`](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/erc4626/ERC4626Oracle.sol#L35-L43) works properly only if the number of decimals is the same, but no check is in place to revert when the number of decimals is different. The `OracleFacade` `admin` could inadvertently or unknowingly add a `ERC4626` with different `decimals`.

## Impact
If decimals of `asset` < decimals of `ERC4626` token, then the price is underestimated. When decimals of `asset` > decimals of `ERC4626` token, then the price is overestimated, leading to a collateral calculation for the account that is overestimated by orders of magnitude. The account can then withdraw huge amounts of funds.
An example: 

`decimals` of `ERC4626` : 8

`decimals` of `asset` : 18

assume the exchange ratio between `ERC4626` and `asset` : 1 to 1, 
so `IERC4626(token).previewRedeem(10**8)` is equal to `10**18`.

the price of `10**8` `shares` in `wei` is: `10**18 * oracleFacade.getPrice(IERC4626(token).asset()) / 10**8` = `10**10 * oracleFacade.getPrice(IERC4626(token).asset())`, so `10**10` times bigger than expected. This allows an account holding the `ERC4626` as a collateral to withdraw up to `10**10` more funds than expected.

## Code Snippet

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/erc4626/ERC4626Oracle.sol#L35-L43

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

## Tool used

Manual Review

## Recommendation
Adjust the price calculation accordingly:
``` diff
    function getPrice(address token) external view returns (uint) {
        uint decimals = IERC4626(token).decimals();
        return IERC4626(token).previewRedeem(
            10 ** decimals
        ).mulDivDown(
            oracleFacade.getPrice(IERC4626(token).asset()),
-           10 ** decimals
+           10 ** IERC4626(token).asset().decimals()
        );
    }
```

Alternatively, if only equal decimals `ERC4626` must be supported, check and revert accordingly:
```solidity 
require(IERC4626(token).decimals() == IERC4626(token).asset().decimals(), "ERC4626 tokens with decimals different from underlying not supported")
```