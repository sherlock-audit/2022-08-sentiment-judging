WATCHPUG
# Lack of resolution plan for insolvent accounts may lead to permanent bad debts and then a bank run on LToken

## Summary

A flash crash of the collateral asset can suddenly cause the account to become insolvent (`balanceToBorrow <1`). In the current design/implementation, these accounts cannot be liquidated.

## Vulnerability Detail

With the current implementation, the liquidator must repay all the `accountBorrows` in full to liquidate a unhealthy account (`balanceToBorrow` < 1.2).

This will work in most cases.

However, under extreme market conditions (plus price oracle delay), the value of the account's assets can be lower than the borrowed amounts.

In that case, these accounts will not be liquidated unless the value of the assets rises and the accounts become solvent again.

When one or more accounts go insolvent (`balanceToBorrow < 1`), the bad debt will be accumulated in the `LToken`, since a portion of the `borrows` on the `LToken` will never be repaid (either through liquidation or repay by the user).

In other words, the actual total assets are now less than the total liabilities.

However, in the current implementation, when the user withdraws, the shortfall is not taken into consideration, so the early users can get all their money back as there are still enough assets for them to exit, but the late users or at least the last user won't be able to withdraw all their balance.

This will make the LP of `LToken` rush to withdraw their balances, aka a bank run.

## PoC

1. Alice deposits 2 ETH and borrows 10 ETH worth of BTC;
2. Alice swaps all BTC and ETH to LUNA, got 12 ETH worth of LUNA;
3. LUNA/ETH price down 10%, BTC/ETH price up 8%, Alice now have 10.8 ETH worth assets and 10.8 ETH worth debt, since there is no profit, no liquidator will liquidate Alice's account.

As the price of LUNA/ETH continues to drop, the 10 ETH bad debt can never be repaid to LToken.

## Impact

The accrual of long term or even permanent bad debt, which will further cause bank run of the LTokens.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L367-L385

## Tool used

Manual Review

## Recommendation

Consider introducing a collateral auction or allow partial repayment:

#### MakerDAO Collateral Auction

> In the context of the Maker protocol, a liquidation is the automatic transfer of collateral from an insufficiently collateralized Vault, along with the transfer of that Vault’s debt to the protocol. In the liquidation contract (the Dog), an auction is started promptly to sell the transferred collateral for DAI in an attempt to cancel out the debt now assigned to the protocol.

Ref: https://docs.makerdao.com/smart-contract-modules/dog-and-clipper-detailed-documentation

#### Aave Protocol Liquidations

> The health of the Aave Protocol is dependent on the 'health' of the collateralised positions within the protocol, also known as the 'health factor'. When the 'health factor' of an account's total loans is below 1, anyone can make a `liquidationCall()` to the Pool or L2Pool (in case of Arbitrum/Optimism) contract, pay back part of the debt owed and receive discounted collateral in return (also known as the liquidation bonus).

Ref: https://docs.aave.com/developers/guides/liquidations

#### Compound III Liquidation

> When an account’s borrow balance exceeds the limits set by liquidation collateral factors, it is eligible for liquidation. A liquidator (a bot, contract, or user) can call the absorb function, which relinquishes ownership of the accounts collateral, and returns the value of the collateral, minus a penalty (liquidationFactor), to the user in the base asset. The liquidated user has no remaining debt, and typically, will have an excess (interest earning) balance of the base asset.

Ref: https://docs.compound.finance/liquidation/#liquidation