WATCHPUG
# `ERC4626Oracle` Price will be wrong when the ERC4626's `decimals` is different from the underlying token’s decimals 

## Summary

EIP-4626 does not require the decimals must be the same as the underlying tokens' decimals, and when it's not, `ERC4626Oracle` will malfunction.

## Vulnerability Detail

In the current implementation, `IERC4626(token).decimals()` is used as the `IERC4626(token).asset()`'s decimals to calculate the ERC4626's price.

However, while most ERC4626s are using the underlying token’s decimals as `decimals`, there are some ERC4626s use a different decimals from underlying token’s decimals since EIP-4626 does not require the decimals must be the same as the underlying token’s decimals:

> Although the convertTo functions should eliminate the need for any use of an EIP-4626 Vault’s decimals variable, it is still strongly recommended to mirror the underlying token’s decimals if at all possible, to eliminate possible sources of confusion and simplify integration across front-ends and for other off-chain users.

Ref: https://eips.ethereum.org/EIPS/eip-4626

## Impact

The price of ERC4626 will be significantly underestimated when the underlying token's decimals > ERC4626's decimals, and be significantly overestimated when the underlying token's decimals < ERC4626's decimals.

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

`getPrice()` can be changed to:

```solidity
    function getPrice(address token) external view returns (uint) {
        uint decimals = IERC4626(token).decimals();
        address underlyingToken = IERC4626(token).asset();
        return IERC4626(token).previewRedeem(
            10 ** decimals
        ).mulDivDown(
            oracleFacade.getPrice(underlyingToken),
            10 ** IERC20Metadata(underlyingToken).decimals()
        );
    }
```