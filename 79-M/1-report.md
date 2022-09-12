hyh
# LEther's redeemEth can burn shares of a user for nothing

## Summary    

A user can lose all his ETH held with the LEther via sequence of dust redeemEth() calls, with each burning some shares, but yielding zero ETH for the user due to rounding down in previewRedeem().

## Vulnerability Detail

previewRedeem() and previewDeposit() call convertToAssets() and convertToShares() correspondingly, with both functions performing `mulDivDown`, so the resulting amount can be rounded to zero. Contrary to other parts of the implementation this isn't checked in redeemEth(), so it can lose all user's funds with a sequence of `redeemEth(shares)` with small enough `shares`, each of them rounding to zero `assets`.

## Impact

User can lose up to all funds held with the LEther due to rounding. I.e. the sequence of redeemEth() calls can empty a LEther account of any size.

Placing severity to be **medium** as that's contingent on user actions, each of them being, however, fully valid. Theoretically this can be used as a base of griefing attack by a third party, which, for example, not being able to directly retrieve funds, but can make a user lose all LEther held funds with such a calls.

## Code Snippet

`assets` in redeemEth() aren't controlled to be non-zero after rounding:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LEther.sol#L47-L49

previewRedeem() rounds down:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/utils/ERC4626.sol#L154-L156

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/utils/ERC4626.sol#L132-L136

## Recommendation

Consider adding the check similar to other parts of the logic:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LEther.sol#L47-L49

```solidity
    function redeemEth(uint shares) external {
        uint assets = previewRedeem(shares);
+       require(assets != 0, "ZERO_ASSETS");
        _burn(msg.sender, shares);
```

As a broader measure consider introducing dust threshold and require that it should be met for the each amount retrieved.

The threshold value should correspond to the situation when it's not economically feasible to spend gas on the call given some historical median price of it.