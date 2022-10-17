WATCHPUG
# A malicious early user/attacker can manipulate the LToken's pricePerShare to take an unfair share of future users' deposits

## Summary

A well known attack vector for almost all shares based liquidity pool contracts, where an early user can manipulate the price per share and profit from late users' deposits because of the precision loss caused by the rather large value of price per share.

## Vulnerability Detail

A malicious early user can `deposit()` with `1 wei` of `asset` token as the first depositor of the LToken, and get `1 wei` of shares.

Then the attacker can send `10000e18 - 1` of `asset` tokens and inflate the price per share from 1.0000 to an extreme value of 1.0000e22 ( from `(1 + 10000e18 - 1) / 1`) .

As a result, the future user who deposits `19999e18` will only receive `1 wei` (from `19999e18 * 1 / 10000e18`) of shares token.

They will immediately lose `9999e18` or half of their deposits if they `redeem()` right after the `deposit()`.

## Impact

The attacker can profit from future users' deposits. While the late users will lose part of their funds to the attacker.

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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L138-L140

```solidity
function previewDeposit(uint256 assets) public view virtual returns (uint256) {
    return convertToShares(assets);
}
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L126-L131

```solidity
function convertToShares(uint256 assets) public view virtual returns (uint256) {
    uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

    return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
}
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L191-L193

```solidity
function totalAssets() public view override returns (uint) {
    return asset.balanceOf(address(this)) + getBorrows() - getReserves();
}
```

## Tool used

Manual Review

## Recommendation

Consider requiring a minimal amount of share tokens to be minted for the first minter, and send a port of the initial mints as a reserve to the DAO so that the pricePerShare can be more resistant to manipulation.

```solidity
function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
    beforeDeposit(assets, shares);

    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    // for the first mint, we require the mint amount > (10 ** decimals) / 100
    // and send (10 ** decimals) / 1_000_000 of the initial supply as a reserve to DAO
    if (totalSupply == 0 && decimals >= 6) {
        require(shares > 10 ** (decimals - 2));
        uint256 reserveShares = 10 ** (decimals - 6);
        _mint(DAO, reserveShares);
        shares -= reserveShares;
    }

    // Need to transfer before minting or ERC777s could reenter.
    asset.safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);
}

function mint(uint256 shares, address receiver) public virtual returns (uint256 assets) {
    beforeDeposit(assets, shares);

    assets = previewMint(shares); // No need to check for rounding error, previewMint rounds up.

    // for the first mint, we require the mint amount > (10 ** decimals) / 100
    // and send (10 ** decimals) / 1_000_000 of the initial supply as a reserve to DAO
    if (totalSupply == 0 && decimals >= 6) {
        require(shares > 10 ** (decimals - 2));
        uint256 reserveShares = 10 ** (decimals - 6);
        _mint(DAO, reserveShares);
        shares -= reserveShares;
    }

    // Need to transfer before minting or ERC777s could reenter.
    asset.safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);
}
```
## Sentiment Team
Fixed as recommened but instead of sending these shares to the DAO, we burn them. PR [here](https://github.com/sentimentxyz/protocol/pull/232).

## WatchPug
Confirmed fix. 
