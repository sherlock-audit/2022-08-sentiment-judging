WATCHPUG
# Allowing the admin to update/remove an LToken is dangerous and error-prone as the existing LToken has important accounting storage variables

## Summary

Admin can update and remove LTokens with `setLToken()`, this is a very dangerous and error-prone operation that should not be allowed.

## Vulnerability Detail

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95-L104

```solidity
function setLToken(address underlying, address lToken) external adminOnly {
    if (LTokenFor[underlying] == address(0)) {
        if (lToken == address(0)) revert Errors.ZeroAddress();
        lTokens.push(lToken);
    }
    else if (lToken == address(0)) removeLToken(LTokenFor[underlying]);
    else updateLToken(LTokenFor[underlying], lToken);

    LTokenFor[underlying] = lToken;
}
```

L100-101, the existing LToken can be updated or removed without any checks and constraints. 

## Impact

When an LToken with existing loans gets removed or updated, the borrowed amounts will no longer be included in the user's `totalBorrows`:

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L163-L176

```solidity
function _getBorrows(address account) internal view returns (uint) {
    if (IAccount(account).hasNoDebt()) return 0;
    address[] memory borrows = IAccount(account).getBorrows();
    uint borrowsLen = borrows.length;
    uint totalBorrows;
    for(uint i; i < borrowsLen; ++i) {
        address LTokenAddr = registry.LTokenFor(borrows[i]);
        totalBorrows += _valueInWei(
            borrows[i],
            ILToken(LTokenAddr).getBorrowBalance(account)
        );
    }
    return totalBorrows;
}
```

As a result, the user will be able to withdraw more funds than they should, for example:

- Alice borrowed 10k USDC with 2k USDC as collateral;
- Admin removed the LToken for USDC;
- Alice can withdraw 12k USDC as the system now believes Alice has no debt.

## Code Snippet

## Tool used

Manual Review

## Recommendation

1. Consider only allowing remove and update if the LToken is new and there are no balance and no borrows;
2. Instead of removing and updating, consider pausing the LToken for certain features, like in Compound, cToken's `Mint` and `Borrow` can be paused:

https://github.com/compound-finance/compound-protocol/blob/master/contracts/Comptroller.sol#L1049-L1067

```solidity
function _setMintPaused(CToken cToken, bool state) public returns (bool) {
    require(markets[address(cToken)].isListed, "cannot pause a market that is not listed");
    require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
    require(msg.sender == admin || state == true, "only admin can unpause");

    mintGuardianPaused[address(cToken)] = state;
    emit ActionPaused(cToken, "Mint", state);
    return state;
}

function _setBorrowPaused(CToken cToken, bool state) public returns (bool) {
    require(markets[address(cToken)].isListed, "cannot pause a market that is not listed");
    require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
    require(msg.sender == admin || state == true, "only admin can unpause");

    borrowGuardianPaused[address(cToken)] = state;
    emit ActionPaused(cToken, "Borrow", state);
    return state;
}
```