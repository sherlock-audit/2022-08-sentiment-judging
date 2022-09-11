grhkm
# LEther doesn't update the state on deposits and redeems

## Summary
When depositing or redeeming through the LEther contract the `updateState()` function isn't called. Thus, the interest isn't accounted for and the withdrawal/deposit happens with out-of-date data.

## Vulnerability Detail
`updateState()` increases the borrow and reserve amount by adding the accrued interest. When a user deposits with the old state, they get more shares than they should because the total amount of assets is less than it should be. If they redeem their shares, they get fewer tokens than they should because again, the amount of assets the pool has is higher than the one used for the withdrawal computation.

Let's say $lastUpdated = x$ and a user deposits at $y$ where $y > x$. Any interest that accrued between $x$ and $y$ is not accounted for. Since the rate model computes the interest by the second, this applies to the smallest timeframe which is between two blocks, i.e. 13 seconds. So as long as nobody updated the state in the same block that the user deposited or withdrew their funds, the issue is present. Because of that, I rate this as HIGH. There is no cost to the attack and funds are lost.

## Impact
Lost funds for depositors of the lending pool.

## Code Snippet

Custom logic that skips the call to `updateState()`:
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LEther.sol#L31-L53
```sol
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

In LToken, the deposit and withdrawal are handled by the underlying ERC4626 contract's function. There, `updateState()` is triggered by calling the hooks `beforeDeposit()` and `beforeWithdraw()`:
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L49
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L111


## Tool used

Manual Review

## Recommendation
Add the call to `beforeDeposit()` and `beforeWithdraw()` to the LEther functions.