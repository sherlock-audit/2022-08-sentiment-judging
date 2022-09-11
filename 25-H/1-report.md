grhkm
# ERC4626Oracle: Assumes that asset and share have same number of decimals

## Summary
If the asset and share of a EIP-4626 vault has a different number of decimals than the vault itself, the value that is returned by `getPrice()` will be wrong.

## Vulnerability Detail
In `getPrice()`, the number of decimals of the EIP-4626 share is used to query how much assets will be returned for 1 share (which is fine). But this value is also used to normalize the result that is returned by `previewRedeem`. While this works fine when the number of decimals between the asset and share are identical, the value will be wrong when they differ. `previewRedeem` will then return a value with the decimals of asset (because it returns how much of asset one will get).

It is also mentioned in the standard that there is no need to use the vault's decimals:
> Although the convertTo functions should eliminate the need for any use of an EIP-4626 Vault’s decimals variable, it is still strongly recommended to mirror the underlying token’s decimals if at all possible, to eliminate possible sources of confusion and simplify integration across front-ends and for other off-chain users.

## Impact
In the scenarios mentioned above, the price that is used for the ERC4626 tokens will be either way too high or low (depending on how the decimals differ), meaning that the health factor calculation no longer work correctly.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/erc4626/ERC4626Oracle.sol#L41

## Tool used

Manual Review

## Recommendation
```
        uint decimals = IERC4626(token).decimals();
        return IERC4626(token).previewRedeem(
            10 ** decimals
        ).mulDivDown(
            oracleFacade.getPrice(IERC4626(token).asset()),
            10 ** IERC4626(token).asset().decimals()
        );
```