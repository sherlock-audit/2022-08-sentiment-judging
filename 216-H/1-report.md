__141345__
# External liquidation to steal fund

External liquidation to steal fund

## Summary

Some protocols have liquidation mechanism, which can take fund out of sentiment account. This could be abused to drain sentiment pools.


## Vulnerability Detail

A malicious user can do the following:
1. deposit 1,000 DAI and borrow 500 DAI, total 1,500 DAI in account
2. go to compound with the 1,500 DAI, and borrow DAI to the limit. Assuming LTV is 50%, at the end there would be 3,000 DAI collateral, 1,500 DAI borrow balance, 0 liquidity. (This can be achieved by front deposit 3,000 DAI, borrow 1,500 DAI and withdraw to the limit, or just re-collateral with the borrowed amount).
3. wait for several blocks, let the interest in compound accrue.
4. Due to the interest, the compound account will trigger liquidation threshold, the user can liquidate self with another address, repay $1,500 DAI and get the collateral back (a little less than 1,500 DAI, deducting some liquidation fee)
5. the user profits nearly 500 DAI, sentiment DAI pool loses 500 DAI loan amount.

Even the external lending protocol only liquidate part of the collateral, the loss to sentiment pool will be the same. Because the collateral left will be much lower than the borrow amount, as a consequence, the liquidators won't repay sentiment.

The key point is, there is some way that external contracts can take fund out from the sentiment account.


## Impact

The protocol fund will be drained, unable to operate due to lack of fund.


## Code Snippet

The malicious user will call the following functions:
```solidity
// protocol\src\core\AccountManager.sol

    // deposit 1,000 DAI
    function deposit(address account, address token, uint amt)
        external
        whenNotPaused
        onlyOwner(account)
    {}

    // borrow 500 DAI
    function borrow(address account, address token, uint amt) {}

    // go to compound and borrow to the limit
    function exec(
        address account,
        address target,
        uint amt,
        bytes calldata data
    )
        external
        onlyOwner(account)
    {}

    // finally
    // self liquidate in compound to take out all the fund
    // wait several blocks to accrue the interest
```


## Tool used

Manual Review

## Recommendation

Do not allow assets which could be liquidated, such as tokens from lending protocol (compound, AAVE, etc).

## Comment

The borrow call is not whitelisted in aave and compound