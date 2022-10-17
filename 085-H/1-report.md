WATCHPUG
# `updateState()` should be called in `depositEth()` and `redeemEth()`

## Summary

Whenever the liquidity of a LToken changes, `getRateFactor` will be changed.

Therefore, `updateState()` must be called prior to the change to settle the pending interests.

## Vulnerability Detail

Alice is a liquidity provider for LEther, Bob is a borrower.

1. Alice added 1000 ETH;
2. Bob borrowed 500 ETH; In which `updateState()` is called. The `util` in `getBorrowRatePerSecond` is `0.5` now.
2. One year later (no one interacts with the `asset` for 1 year), Alice redeemed 500 ETH with `redeemEth()`, in which `updateState()` is not called. The `util` in `getBorrowRatePerSecond` is `1` now.
3. Bob called `repay()`, in which `updateState()` is called to calculates the pending interest:

For Bob the borrower, the sum of principal and interest is:

$$
Borrow Rate Per Second = c3 \cdot (util \cdot c1 + util^{32} \cdot c1 + util^{64} \cdot c2) \div secsPerYear = 5545529241
$$

$$
rateFactor = Borrow Rate Per Second \cdot secsPerYear \div 1e18 = 175000000081490720
$$

$$
sum = borrow \cdot rateFactor \div 1e18 + borrow = 587
$$

But the actual sum is as below, due to `updateState()` is not called in `redeemEth()`:

$$
Borrow Rate Per Second = c3 \cdot (util \cdot c1 + util^{32} \cdot c1 + util^{64} \cdot c2) \div secsPerYear = 55455292386
$$

$$
rateFactor = Borrow Rate Per Second \cdot secsPerYear \div 1e18 = 1750000000000000000
$$

$$
sum = borrow \cdot rateFactor \div 1e18 + borrow = 1375
$$


As a result, Bob the borrower is now paying `1375` instead of `587` for the interest, which is 2x the expected amount.

On the other hand, if another liquidity provider called `depositEth()` before Bob repays the loan, the actual interest can be lower than expected, which constitutes a loss of yields to Alice.

## Impact

Incorrect amounts of interests will be paid by the borrowers, which can result in loss of yields to the lenders or overpaid interest for the borrowers.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L200-L227

```solidity
    function updateState() public {
        if (lastUpdated == block.timestamp) return;
        uint rateFactor = getRateFactor();
        uint interestAccrued = borrows.mulWadUp(rateFactor);
        borrows += interestAccrued;
        reserves += interestAccrued.mulWadUp(reserveFactor);
        lastUpdated = block.timestamp;
    }

    /* -------------------------------------------------------------------------- */
    /*                             INTERNAL FUNCTIONS                             */
    /* -------------------------------------------------------------------------- */

    /**
        @dev Rate Factor = Timestamp Delta * 1e18 (Scales timestamp delta to 18 decimals) * Interest Rate Per Block
            Timestamp Delta = Number of seconds since last update
    */
    function getRateFactor() internal view returns (uint) {
        return (block.timestamp == lastUpdated) ?
            0 :
            ((block.timestamp - lastUpdated)*1e18)
            .mulWadUp(
                rateModel.getBorrowRatePerSecond(
                    asset.balanceOf(address(this)),
                    borrows
                )
            );
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/DefaultRateModel.sol#L51-L77

```solidity
function getBorrowRatePerSecond(
        uint liquidity,
        uint borrows
    )
        external
        view
        returns (uint)
    {
        uint util = _utilization(liquidity, borrows);
        return c3.mulDivDown(
            (
                util.mulWadDown(c1)
                + util.rpow(32, SCALE).mulWadDown(c1)
                + util.rpow(64, SCALE).mulWadDown(c2)
            ),
            secsPerYear
        );
    }

    function _utilization(uint liquidity, uint borrows)
        internal
        pure
        returns (uint)
    {
        uint totalAssets = liquidity + borrows;
        return (totalAssets == 0) ? 0 : borrows.divWadDown(totalAssets);
    }
```


`updatestate` must be called everytime balance of asset is changed.
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LEther.sol#L26-L53

```solidity
    /**
        @notice Wraps Eth sent by the user and deposits into the LP
            Transfers shares to the user denoting the amount of Eth deposited
        @dev Emits Deposit(caller, owner, assets, shares)
    */
    function depositEth() external payable {
        uint assets = msg.value;
        uint shares = previewDeposit(assets);
        require(shares != 0, "ZERO_SHARES");
        IWETH(address(asset)).deposit{value: assets}();
        _mint(msg.sender, shares);
        emit Deposit(msg.sender, msg.sender, assets, shares);
    }

    /**
        @notice Unwraps Eth and transfers it to the caller
            Amount of Eth transferred will be the total underlying assets that
            are represented by the shares
        @dev Emits Withdraw(caller, receiver, owner, assets, shares);
        @param shares Amount of shares to redeem
    */
    function redeemEth(uint shares) external {
        uint assets = previewRedeem(shares);
        _burn(msg.sender, shares);
        emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
        IWETH(address(asset)).withdraw(assets);
        msg.sender.safeTransferEth(assets);
    }
```

## Tool used

Manual Review

## Recommendation

`beforeDeposit()` should be called in `depositEth()` and `redeemEth()`:

```solidity
    /**
        @notice Wraps Eth sent by the user and deposits into the LP
            Transfers shares to the user denoting the amount of Eth deposited
        @dev Emits Deposit(caller, owner, assets, shares)
    */
    function depositEth() external payable {
        uint assets = msg.value;
        uint shares = previewDeposit(assets);
        require(shares != 0, "ZERO_SHARES");
        beforeDeposit(assets, shares);
        IWETH(address(asset)).deposit{value: assets}();
        _mint(msg.sender, shares);
        emit Deposit(msg.sender, msg.sender, assets, shares);
    }

    /**
        @notice Unwraps Eth and transfers it to the caller
            Amount of Eth transferred will be the total underlying assets that
            are represented by the shares
        @dev Emits Withdraw(caller, receiver, owner, assets, shares);
        @param shares Amount of shares to redeem
    */
    function redeemEth(uint shares) external {
        uint assets = previewRedeem(shares);
        beforeWithdraw(assets, shares);
        _burn(msg.sender, shares);
        emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
        IWETH(address(asset)).withdraw(assets);
        msg.sender.safeTransferEth(assets);
    }
```
## Sentiment Team
Fixed as recommended. PR [here](https://github.com/sentimentxyz/protocol/pull/230).

## Lead Senior Watson
Confirmed fix. 
