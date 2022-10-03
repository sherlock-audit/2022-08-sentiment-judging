cccz
# LEther:  beforeDeposit()/beforeWithdraw()  should be called at the beginning of  depositEth()/redeemEth()

## Summary
LEther: beforeDeposit()/beforeWithdraw()  should be called at the beginning of  depositEth()/redeemEth()
## Vulnerability Detail
In the depositEth()/redeemEth() functions of the LEther contract, in order to use the latest results of totalAssets(), beforeDeposit()/beforeWithdraw() should be called before calling previewDeposit()/previewRedeem(). 
The updateState() in beforeDeposit()/beforeWithdraw() will update the borrows and reserves variables so that the results of totalAssets() are the latest.
## Impact
Since the depositEth()/redeemEth() function of the LEther contract does not call the updateState() function, the totalAssets() used in the depositEth()/redeemEth() function will be smaller. It will cause the share minted in depositEth() to increase, and the assets withdrawn in redeemEth() to decrease.
## Code Snippet
```
    function depositEth() external payable {
        uint assets = msg.value;
        uint shares = previewDeposit(assets);
        require(shares != 0, "ZERO_SHARES");
        IWETH(address(asset)).deposit{value: assets}();
        _mint(msg.sender, shares);
        emit Deposit(msg.sender, msg.sender, assets, shares);
    }
    function redeemEth(uint shares) external {
        uint assets = previewRedeem(shares);
        _burn(msg.sender, shares);
        emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
        IWETH(address(asset)).withdraw(assets);
        msg.sender.safeTransferEth(assets);
    }
...
    function beforeDeposit(uint, uint) internal override { updateState(); }
    function beforeWithdraw(uint, uint) internal override { updateState(); }
```
## Tool used

Manual Review

## Recommendation
Change to
```
    function depositEth() external payable returns (uint256 shares){
        uint assets = msg.value;
        beforeDeposit(assets, shares);
        shares = previewDeposit(assets);
        require(shares != 0, "ZERO_SHARES");
        IWETH(address(asset)).deposit{value: assets}();
        _mint(msg.sender, shares);
        emit Deposit(msg.sender, msg.sender, assets, shares);
    }
    function redeemEth(uint shares) external returns (uint256 assets){
        beforeWithdraw(assets, shares);
        assets = previewRedeem(shares);
        _burn(msg.sender, shares);
        emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
        IWETH(address(asset)).withdraw(assets);
        msg.sender.safeTransferEth(assets);
    }
```