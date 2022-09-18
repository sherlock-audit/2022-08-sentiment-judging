hansfriese
# In `LToken.sol`, it doesn't convert assets to shares(shares to assets) for the protocol's advantage.

## Summary
In `LToken.sol`, it doesn't convert assets to shares(shares to assets) for the protocol's advantage.


## Vulnerability Detail
https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L159

https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L236


## Impact
In `LToken.sol`, it doesn't convert assets to shares(shares to assets) for the protocol's advantage.

When borrowers repay the loan, there would be some loss to the protocol because of the division rounding issue.


## Proof of Concept
Currently, both functions `lendTo()` and `collectFrom()` use the same function `convertAssetToBorrowShares()` to convert asset to shares.

And `convertAssetToBorrowShares()` rounds up after division.

```
function convertAssetToBorrowShares(uint amt) internal view returns (uint) {
    uint256 supply = totalBorrowShares;
    return supply == 0 ? amt : amt.mulDivUp(supply, getBorrows());
}
```

Normally such rounding should be calculated for the protocol's advantage and it should round down for `collectFrom()`.

Also, `convertBorrowSharesToAsset()` should round up because this function is used when borrowers repay.


## Tool used
Manual Review


## Recommendation
I think we can modify 2 functions like below.

```
function convertAssetToBorrowShares(uint amt, bool roundUp) internal view returns (uint) {
    uint256 supply = totalBorrowShares;
    return supply == 0 ? amt : (roundUp ? amt.mulDivUp(supply, getBorrows()) : amt.mulDivDown(supply, getBorrows()));
}

function convertBorrowSharesToAsset(uint debt) internal view returns (uint) {
    uint256 supply = totalBorrowShares;
    return supply == 0 ? debt : debt.mulDivUp(getBorrows(), supply);
}
```

And `roundUp = true` should be used for `lendTo()`, `roundUp = false` should be used for `collectFrom()`.