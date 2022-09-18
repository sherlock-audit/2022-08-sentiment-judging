rbserver
# `deposit`, `mint`, `withdraw`, and `redeem` functions in `ERC4626` contract can be called with `receiver` input being `address(0)`

## Summary
When the `deposit`, `mint`, `withdraw`, and `redeem` functions in the `ERC4626` contract are called with the `receiver` input being `address(0)`, the minted `LToken` shares and transferred asset amounts are lost.

## Vulnerability Detail
As shown in the Code Snippet section, when calling the `deposit`, `mint`, `withdraw`, and `redeem` functions in the `ERC4626` contract, the `receiver` `address` input is not checked against `address(0)`. If `address(0)` is used as `receiver`, the `LToken` shares can be minted to `address(0)` when calling `deposit` or `mint`, and the asset amount can be transferred to `address(0)` when calling `withdraw` or `redeem`. These shares and asset amounts would be lost because no one controls `address(0)`.

## Impact
It is possible that users can accidentally provide `address(0)` for the address inputs, especially if the relevant frontend malfunctions or is hacked. When this happens, because the sanity checks against `address(0)` are missing, these minted `LToken` shares and transferred asset amounts would be owned by `address(0)` after calling these `deposit`, `mint`, `withdraw`, and `redeem` functions. As a result, these shares and asset amounts are lost forever.

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L48-L60
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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L62-L73
```solidity
    function mint(uint256 shares, address receiver) public virtual returns (uint256 assets) {
        beforeDeposit(assets, shares);

        assets = previewMint(shares); // No need to check for rounding error, previewMint rounds up.

        // Need to transfer before minting or ERC777s could reenter.
        asset.safeTransferFrom(msg.sender, address(this), assets);

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L75-L95
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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L97-L118
```solidity
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public virtual returns (uint256 assets) {
        if (msg.sender != owner) {
            uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.

            if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
        }

        // Check for rounding error since we round down in previewRedeem.
        require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");

        beforeWithdraw(assets, shares);

        _burn(owner, shares);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);

        asset.safeTransfer(receiver, assets);
    }
```

## Tool used
Manual Review

## Recommendation
In each of these `deposit`, `mint`, `withdraw`, and `redeem` functions, a statement can be added so the function call will revert if the `receiver` input is `address(0)`.