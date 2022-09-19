WATCHPUG
# Lack of Debt Floor can result in insolvent smaller accounts won't be liquidated

## Summary

A Debt Floor parameter is needed to prevent users from creating multiple vaults with very low debt amount and collateral. Otherwise, the liquidators may be reluctant to liquidate smaller vaults because the reward is low in comparison to the gas costs of liquidation, which can further result in the accrual of bad debt.

## Vulnerability Detail

The current system allows accounts with arbitrary debt and collateral amounts, which can be as small as $0.1 or even 1 Wei in ETH.

This can be a problem as the liquidators may be reluctant to liquidate smaller vaults because the reward is low in comparison to the gas costs of liquidation.

MakerDAO has the Debt Floor parameter which controls the minimum amount of DAI that can be minted using a specific vault type for an individual vault:

https://makerdao.world/en/learn/governance/param-debt-floor/

## Impact

The accrual of bad debt.

### PoC

1. Alice deposited `$10` worth of BTC as collateral and borrowed `$40` of ETH;
2. BTC's price dropped by 10% and ETH price has raised by 10% so that the account become liquidatable, the total assets is now `$51` and the borrowed amount is `$44`; However, as the gas cost to liquidate the account is higher than the potential gain: `$7`, the account will not be liquidated.

If the price can not be restored, this will result in the accrual of bad debt.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L203-L217

```solidity
function borrow(address account, address token, uint amt)
    external
    whenNotPaused
    onlyOwner(account)
{
    if (registry.LTokenFor(token) == address(0))
        revert Errors.LTokenUnavailable();
    if (!riskEngine.isBorrowAllowed(account, token, amt))
        revert Errors.RiskThresholdBreached();
    if (IAccount(account).hasAsset(token) == false)
        IAccount(account).addAsset(token);
    if (ILToken(registry.LTokenFor(token)).lendTo(account, amt))
        IAccount(account).addBorrow(token);
    emit Borrow(account, msg.sender, token, amt);
}
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L128-L145

```solidity
function lendTo(address account, uint amt)
    external
    whenNotPaused
    accountManagerOnly
    returns (bool isFirstBorrow)
{
    updateState();
    isFirstBorrow = (borrowsOf[account] == 0);

    uint borrowShares;
    require((borrowShares = convertAssetToBorrowShares(amt)) != 0, "ZERO_BORROW_SHARES");
    totalBorrowShares += borrowShares;
    borrowsOf[account] += borrowShares;

    borrows += amt;
    asset.safeTransfer(account, amt);
    return isFirstBorrow;
}
```

## Tool used

Manual Review

## Recommendation

Consider adding a `Debt Floor` parameter to control the minimum amount borrowed by one account.

Likewise, if a user attempts to pay back debt such that their debt will equal less than the Debt Floor and greater than zero, the transaction should revert.