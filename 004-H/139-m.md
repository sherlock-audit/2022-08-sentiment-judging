berndartmueller
# First `ERC4626` deposit can break share calculation

## Summary

The first depositor of an `ERC4626` (`LToken`) vault can maliciously manipulate the share price by depositing the lowest possible amount (`1 wei`) of liquidity and then artificially inflating `ERC4626.totalAssets`.

This can inflate the base share price as high as 1:1e18 early on, which force all subsequence deposit to use this share price as a base and worst case, due to rounding down, if this malicious initial deposit front-run someone else depositing, this depositor will receive 0 shares and lost his deposited assets.

## Vulnerability Detail

Given an `LToken` ERC4626 vault with `DAI` as the underlying asset:

1. Alice (attacker) deposits initial liquidity of `1 wei` `DAI` via `ERC4626.deposit`
2. Alice receives `1e18` (1 wei) vault shares
3. Alice transfers `1 ether` of `DAI` via `transfer()` to the vault to artificially inflate the `asset` balance without minting new shares. The `asset` balance is now `1 ether + 1 wei` DAI -> vault share price is now very high (= `1000000000000000000001` wei ~ `1000 * 1e18`)
4. Bob (victim) deposits `100 ether` `DAI`
5. Bob receives `0` shares

Bob receives `0` shares due to a precision issue. His deposited funds are lost.

The shares are calculated as following ([ERC4626.sol#L129](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L129)):

```solidity
return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
```

In case of a very high share price, due to `totalAssets()` `>` `assets * supply`, shares will be 0.

## Impact

`ERC4626` vault share price can be maliciously inflated on the initial deposit, leading to the next depositor losing assets due to precision issues.

## Code Snippet

See explanation above

## Tools Used

Manual review

## Recommendation

This is a well-known issue, Uniswap and other protocols had similar issues when `supply == 0`.

For the first deposit, mint a fixed amount of `shares`, e.g. `10**decimals()`

```solidity
if (supply == 0) {
    return 10**decimals; // @audit-info recommended mitigation
} else {
    return assets.mulDivDown(supply, totalAssets());
}
```

Fix inspired by [Uniswap V2](https://github.com/Uniswap/v2-core/blob/4dd59067c76dea4a0e8e4bfdda41877a6b16dedc/contracts/UniswapV2Pair.sol#L120-L121) and [Harvest Finance](https://github.com/harvest-finance/harvest/blob/master/contracts/Vault.sol#L150)
