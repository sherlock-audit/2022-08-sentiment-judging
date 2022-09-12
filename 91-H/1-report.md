PwnPatrol
# First depositor into each LToken can break share calculation and steal funds

## Summary

The first depositor into any new LToken can buy a small number of shares and send a large amount of the underlying asset to the contract. This will break the share calculation and make shares very expensive for the next depositors. 

Since the deposit function rounds down based on share price, this will cause future depositors to overpay for their shares, accruing additional value to the depositors initial share, which they can then extract as profit.

## Vulnerability Detail

The `deposit()` function calculates the number of shares by calling `previewDeposit(assets)`, which calls `converToShares(asset)`, which performs the following calculation:

`return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());`

If the first depositor is to deposit 1 wei of the asset, they will receive 1 share of the protocol.

They then proceed to send a much larger number, say 10 ** 18, of the underlying token directly to the LToken address.

When the next depositor arrives, supply = 1 and totalAssets = 10 ** 18. This will have the effect of rounding their purchase down to the nearest 10 ** 18, leaving the remainder of assets in the protocol. For example, if they deposit 1.9 * 10 ** 18 tokens, they will receive `1.9 * 10 ** 18 (assets) * 1 (supply) / 10 ** 18 (totalAssets) = 1 share`.

Then original depositor can then call `withdraw()` with a target withdrawal of 1.45 * 10 ** 18. The `previewWithdraw()` function calculates `1.45 * 10 ** 18 (assets to withdraw) * 2 (supply) / 2.9 * 10 ** 19 (totalAssets) = 1 share`, burning their single share and capturing them 0.45 * 10 ** 18 in profit.

## Impact

The first depositor is able to massively inflate the price of the shares, causing rounding down errors for future depositors, and stealing a portion of any excess funds deposited.

## Code Snippet

```solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
    beforeDeposit(assets, shares);

    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    // Need to transfer before minting or ERC777s could reenter.
    asset.safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);
}
```
```solidity
function convertToAssets(uint256 shares) public view virtual returns (uint256) {
    uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

    return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
}
```
```solidity
function withdraw(
    uint256 assets,
    address receiver,
    address owner
) public virtual returns (uint256 shares) {
    shares = previewWithdraw(assets); // No need to check for rounding error, previewWithdraw rounds up.

    if (msg.sender != owner) {
        uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.

        if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
    }

    beforeWithdraw(assets, shares);

    _burn(owner, shares);

    emit Withdraw(msg.sender, receiver, owner, assets, shares);

    asset.safeTransfer(receiver, assets);
}
```
```solidity
function previewMint(uint256 shares) public view virtual returns (uint256) {
    uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

    return supply == 0 ? shares : shares.mulDivUp(totalAssets(), supply);
}
```



## Tool Used

Manual Review

## Recommendation

Add a step to your deployment flow to seed all LTokens with initial liquidity of the underlying token.