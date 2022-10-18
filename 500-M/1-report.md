WATCHPUG

# LToken's implmentation is not fully up to EIP-4626's specification

## Summary

Note: This issue is a part of the extra scope added by Sentiment AFTER the audit contest. This scope was only reviewed by WatchPug and relates to these three PRs:

1. [Lending deposit cap](https://github.com/sentimentxyz/protocol/pull/234)
2. [Fee accrual modification](https://github.com/sentimentxyz/protocol/pull/233)
3. [CRV staking](https://github.com/sentimentxyz/controller/pull/41)

LToken's implmentation is not fully up to EIP-4626's specification. This issue is would actually be considered a Low issue if it were a part of a Sherlock contest. 

## Vulnerability Detail

https://github.com/sentimentxyz/protocol/blob/ccfceb2805cf3595a95198c97b6846c8a0b91506/src/tokens/utils/ERC4626.sol#L185-L187

```solidity
function maxMint(address) public view virtual returns (uint256) {
    return type(uint256).max;
}
```

MUST return the maximum amount of shares mint would allow to be deposited to receiver and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary). This assumes that the user has infinite assets, i.e. MUST NOT rely on balanceOf of asset.

https://eips.ethereum.org/EIPS/eip-4626#:~:text=MUST%20return%20the%20maximum%20amount%20of%20shares,NOT%20rely%20on%20balanceOf%20of%20asset

maxMint() and maxDeposit() should reflect the limitation of maxSupply.

## Impact

Could cause unexpected behavior in the future due to non-compliance with EIP-4626 standard. 

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/ccfceb2805cf3595a95198c97b6846c8a0b91506/src/tokens/utils/ERC4626.sol#L185-L187

```solidity
function maxMint(address) public view virtual returns (uint256) {
    return type(uint256).max;
}
```

## Tool used

Manual Review

## Recommendation

maxMint() and maxDeposit() should reflect the limitation of maxSupply.

Consider changing maxMint() and maxDeposit() to:

```solidity
function maxMint(address) public view virtual returns (uint256) {
    if (totalSupply >= maxSupply) {
        return 0;
    }
    return maxSupply - totalSupply;
}
```

```solidity
function maxDeposit(address) public view virtual returns (uint256) {
    return convertToAssets(maxMint(address(0)));
}
```

## Sentiment Team
Fixed as recommended. PR [here](https://github.com/sentimentxyz/protocol/pull/235).

## Lead Senior Watson
Confirmed fix. 
