cccz
# ERC4626: beforeWithdraw() should be called at the beginning of withdraw()/redeem()

## Summary
ERC4626: beforeWithdraw() should be called at the beginning of withdraw()/redeem()
## Vulnerability Detail
LToken contract's beforeWithdraw() will call updateState() to update the borrows and reserves variables, which will affect the result of totalAssets() and thus the result of previewWithdraw()/previewRedeem().
The user should use the latest borrows and reserves variables to calculate the withdrawn assets, so beforeWithdraw() should be called before previewWithdraw()/previewRedeem()
## Impact
Since the totalAssets() used in withdraw()/redeem() is smaller, the share burned in withdraw() will increase and the assets withdrawn in redeem() will decrease, both of which will cause the user to suffer losses.
## Code Snippet
```
    function beforeWithdraw(uint, uint) internal override { updateState(); }
...
    function updateState() public {
        if (lastUpdated == block.timestamp) return;
        uint rateFactor = getRateFactor();
        uint interestAccrued = borrows.mulWadUp(rateFactor);
        borrows += interestAccrued;
        reserves += interestAccrued.mulWadUp(reserveFactor);
        lastUpdated = block.timestamp;
    }
...
    function totalAssets() public view override returns (uint) {
        return asset.balanceOf(address(this)) + getBorrows() - getReserves(); // @ audit: updateState() increases totalAssets()
    }
...
        require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");

        beforeWithdraw(assets, shares); // @ audit: updateState() is called after totalAssets() is used
...
        shares = previewWithdraw(assets); // No need to check for rounding error, previewWithdraw rounds up.

        if (msg.sender != owner) {
            uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.

            if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
        }

        beforeWithdraw(assets, shares); // @ audit: updateState() is called after totalAssets() is used
...
    function previewWithdraw(uint256 assets) public view virtual returns (uint256) {
        uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

        return supply == 0 ? assets : assets.mulDivUp(supply, totalAssets());
    }
...
    function previewRedeem(uint256 shares) public view virtual returns (uint256) {
        return convertToAssets(shares);
    }
...
    function convertToAssets(uint256 shares) public view virtual returns (uint256) {
        uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

        return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
    }
```
## Tool used

Manual Review

## Recommendation

The user should use the latest borrows and reserves variables to calculate the withdrawn assets, so beforeWithdraw() should be called before previewWithdraw()/previewRedeem()