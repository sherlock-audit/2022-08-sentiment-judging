ellahi
# EIP4626 Rounding error.

## Summary
`convertAssetToBorrowShares` rounding does not conform to EIP4626.
## Vulnerability Detail
Per EIP 4626's Security Considerations (https://eips.ethereum.org/EIPS/eip-4626)

> Finally, ERC-4626 Vault implementers should be aware of the need for specific, opposing rounding directions across the different mutable and view methods, as it is considered most secure to favor the Vault itself during calculations over its users:
> - If (1) it’s calculating how many shares to issue to a user for a certain amount of the underlying tokens they provide or (2) it’s determining the amount of the underlying tokens to transfer to them for returning a certain amount of shares, it should round down.
> - If (1) it’s calculating the amount of shares a user has to supply to receive a given amount of the underlying tokens or (2) it’s calculating the amount of underlying tokens a user has to provide to receive a certain amount of shares, it should round up.

Thus, the result of the `convertAssetToBorrowShares()` should be rounded down.

## Impact
Medium. EIP4626 is aimed to create a consistent and robust implementation patterns for Tokenized Vaults. A slight deviation from 4626 would broke composability and potentially lead to loss of funds. It is counterproductive to implement EIP4626 but does not conform to it fully. 
## Code Snippet
[`LToken.sol`](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L229-L232)
```solidity
function convertAssetToBorrowShares(uint amt) internal view returns (uint) {
    uint256 supply = totalBorrowShares;
    return supply == 0 ? amt : amt.mulDivUp(supply, getBorrows());
}
```
## Tool used

Manual Review

## Recommendation
Ensure that the rounding of vault's functions behave as expected. `convertAssetToBorrowShares()` should round down.
