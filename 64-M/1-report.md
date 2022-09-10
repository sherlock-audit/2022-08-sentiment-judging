cergyk
# If account is not liquidated in time and becomes unprofitable to liquidate, there are no incentives for liquidators to liquidate it, possibly locking the funds borrowed by these accounts forever

## Summary

If any account's health falls below 1.0 due to quick price movement and/or network congestion, it is not profitable for liquidators to liquidate it and thus the funds might be frozen in the account, which will lead to `LToken` lenders being unable to withdraw all funds they have deposited.

## Vulnerability Detail

Currently liquidator pays out full debt for the account it liquidates, receiving all account assets in return. This works when the account health is above 1.0 as it's profitable for the liquidator. However, if liquidators do not liquidate an account in time (due to quick price movement or network congestion), then liquidating account will put liquidators in a loss, so they won't liquidate it. If the price which caused the liquidation never recovers (such as `LUNA` crash), then amount borrowed by the account will be locked in it forever. This will make `LToken` impossible to pay back all the lenders, which can trigger a bank run (the last lenders to exit will not be able to withdraw).

## Impact

If a large account quickly falls below 1.0 health for any reason and then never goes back above 1.0 health again, a large borrowed amount might be locked in it, making it impossible to ever withdraw from LToken for all lenders.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-panprog/blob/6da8a0e43d272eda0d40760cd90c50ed7ce21fee/protocol/src/core/AccountManager.sol#L367-L385

## Tool used

Manual Review

## Recommendation

There must be some mechanism to let the protocol unlock borrows from accounts which are not profitable to liquidate. Since such accounts can only be liquidated for a loss, somebody has to pay for it. Typically, this loss is shared pro-rata between lenders (reduced total assets with the same share supply), but can also be paid from some insurance fund or the treasury.

Consider adding "bad debt" liquidations (make it profitable for liquidators to liquidate accounts with health below 1.0), either sharing the bad debt between all lenders or paying it out from the treasury or some insurance fund. 

If you do implement it, be extra careful for re-entrancy, which can easily put account into low health via exec and then liquidate with bad debt.