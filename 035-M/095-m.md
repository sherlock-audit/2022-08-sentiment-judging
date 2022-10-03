PwnPatrol
# LTokens will not work with fee-on-transfer tokens

## Summary

If fee-on-transfer tokens are added to the protocol, the accounting of LTokens (via ERC4626) will lead to accounting mismatches.

## Vulnerability Detail

In `ERC4626.sol`, each of the deposit and withdrawal functions assume that the amount of the token sent is equal to the amount received. This will cause discrepancies in account because the `assets` variable is the pre-fee amount, whereas the amount that totalAssets is incremented will not include the fee anymore.

## Impact

Much of the security of the protocol depends on the fact that deposits and withdrawals do not alter share price. However, since a deposit will mint X shares but only add X - fee assets to the protocol, any deposit of a fee on transfer token will violate this principle.

As a result, a whole new series of exploits (such as sandwich attacks on deposits and withdrawals, since there is no validation of expected output) becomes possible.

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

## Tool used

Manual Review

## Recommendation

Instead of using `assets` directly, you can subtract the balance of the contract after the transfer from the balance before to find the true difference in `totalAssets`.