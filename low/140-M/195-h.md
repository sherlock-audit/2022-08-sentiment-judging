minhquanym
# Rounding issue in `LToken.collectFrom()` function

## Vulnerability Detail

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L159

Function `LToken.collectFrom()` is called by AccountManager to repay for users. It calculates how much `shares` is reduced when users repay `amt` underlying token. 
```solidity
function collectFrom(address account, uint amt)
    external
    accountManagerOnly
    returns (bool)
{
    uint borrowShares;
    require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
    borrowsOf[account] -= borrowShares;
    totalBorrowShares -= borrowShares;

    borrows -= amt;
    return (borrowsOf[account] == 0);
}
```
In this case, `shares` should be round down to favor Vault. Otherwise, anyone can spam repaying small amount to take benefit and leave Vault in loss. But function `convertAssetToBorrowShares()` is rounding up
```solidity
function convertAssetToBorrowShares(uint amt) internal view returns (uint) {
    uint256 supply = totalBorrowShares;
    return supply == 0 ? amt : amt.mulDivUp(supply, getBorrows());
}
```

## Impact

Instead of repaying all at once, attacker can repay small amounts multiple times to take benefit from rounding. It breaks accounting mechanism and Vault will be in loss which means all lenders will be in loss.

## Proof of Concept

Consider the scenario
1. LToken with underlying token X has `totalBorrowShares = 100`, `getBorrows() = 101`
2. Alice repays `amt = 1` token X and `LToken.collectFrom()` is called. It calculated 
```
borrowShares = convertAssetToBorrowShares(amt) = 1 / 100 * 101 = 1.01
```
But it rounded up to `borrowShares = 2`.
3. Alice can repeatedly repay `amt = 1` so she only need to repay around 50% her debt.

## Tool used

Manual Review

## Recommendation

Consider update rounding mechanism in `collectFrom()` function to round down
