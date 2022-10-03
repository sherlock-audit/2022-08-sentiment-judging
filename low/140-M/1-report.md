berndartmueller
# `ERC4626` contract does not fully conform to `EIP4626`

## Summary

The `ERC4626` contract does not fully conform to the `EIP4626` standard and the `LToken` contract, which implements the `ERC4626` contract, should incorporate the pausable mechanism where required.

## Vulnerability Detail

The `ERC4626` contract implements the `EIP4626` standard ([EIP-4626: Tokenized Vault Standard](https://eips.ethereum.org/EIPS/eip-4626)).

However, according to `EIP4626`, the below-mentioned functions do not fully adhere to the specs.

[ERC4626.maxDeposit](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L162-L164)

```solidity
function maxDeposit(address) public view virtual returns (uint256) {
    return type(uint256).max;
}
```

1. MUST factor in global and user-specific limits, if deposits are entirely disabled (even temporarily) it MUST return 0. `LToken` implements the `ERC4626` contract and has a pause mechanism. Therefore, to be compliant, the paused state should be reflected in the `maxDeposit` function.

[ERC4626.maxMint](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L166-L168)

```solidity
function maxMint(address) public view virtual returns (uint256) {
    return type(uint256).max;
}
```

1. MUST factor in global and user-specific limits, if minting is entirely disabled (even temporarily) it MUST return 0. `LToken` implements the `ERC4626` contract and has a pause mechanism. Therefore, to be compliant, the paused state should be reflected in the `maxMint` function.

[ERC4626.maxWithdraw](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L170-L172)

```solidity
function maxWithdraw(address owner) public view virtual returns (uint256) {
    return convertToAssets(balanceOf[owner]);
}
```

1. MUST factor in global and user-specific limits, if withdrawals are entirely disabled (even temporarily) it MUST return 0. `LToken` implements the `ERC4626` contract and has a pause mechanism. Therefore, to be compliant, the paused state should be reflected in the `maxWithdraw` function.

[ERC4626.maxRedeem](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L174-L176)

```solidity
function maxRedeem(address owner) public view virtual returns (uint256) {
    return balanceOf[owner];
}
```

1. MUST factor in global and user-specific limits, if redemptions are entirely disabled (even temporarily) it MUST return 0. `LToken` implements the `ERC4626` contract and has a pause mechanism. Therefore, to be compliant, the paused state should be reflected in the `maxRedeem` function.

## Impact

Not adhering to the `EIP4626` standard could lead to integration issues with other protocols.

## Code Snippet

See above.

## Tools Used

Manual review

## Recommendation

Consider implementing the `EIP4626` standard as close to the specs as possible.
