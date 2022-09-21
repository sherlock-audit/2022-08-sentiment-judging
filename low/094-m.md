PwnPatrol
# Lenders can get forced into unexpected rates on their deposits and withdrawals

## Summary

In the `ERC4626.sol` contract, each of the functions for depositing or withdrawing assets take in either the assets or shares as an argument, and automatically calculate the corresponding assets or shares for the other side of the trade.

This can lead to situations where users make a deposit or withdrawal and end up with expected results.

## Vulnerability Detail

The `deposit()`, `mint()`, `withdraw()`, and `redeem()` functions in `ERC4626.sol` each take in either the assets or shares, and use the `previewX()` function to calculate the amount of the corresponding asset that should be received.

However, these values have the potential to change rapidly, and users could lose a substantial amount of funds based on these state variables changing before their transaction is processed.

As an example:
- Let's imagine that an LToken currently has 100 assets and 100 shares
- Alice wants to take a major position in the LToken, so she mints 100 shares with the expectation it will cost her 100 assets
- Bob wants to harm Alice, so when he sees her tx in the mempool, he frontruns it with a tx to send 1000 assets directly to the LToken contract
- Alice's tx then mints her 100 shares, but ends up costing her 1100 assets (`100 (shares) * 1100 (totalAssets) / 100 (supply)`)
- If she isn't careful, Alice may not even realize this happened, as her wallet now has 100 shares as expected

## Impact

Lenders can end up with different amounts of assets and shares than expected when they performed their transaction.

## Code Snippet

```solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares);
function mint(uint256 shares, address receiver) public virtual returns (uint256 assets);
function withdraw(uint256 assets, address receiver, address owner) public virtual returns (uint256 shares);
function redeem(uint256 shares, address receiver, address owner) public virtual returns (uint256 assets);
```

## Tool used

Manual Review

## Recommendation

Add an argument to each of these functions to validate the minimal return value at which users are willing to go forward with the transaction. This argument should be compared to the value from state before proceeding.

- `deposit()` should include `minSharesOut`
- `mint()` should include `maxAssetsIn`
- `withdraw()` should include `maxSharesIn`
- `redeem()` should include `minAssetsOut`