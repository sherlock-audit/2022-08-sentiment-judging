hyh
# LToken's and LEther's first depositor can depress shares denomination

## Summary

An attacker can become the first depositor for a recently created LToken and LEther contracts, depressing shares denomination and stealing from all successive depositors.

## Vulnerability Detail

Let's examine the LToken case with DAI as the underlying token. Mike the first depositor can provide a tiny amount of token by calling `deposit(1 wei, Mike)`. Mike then can directly transfer, for example, `10^6 * 1e18 - 1` of the DAI to the LToken contract. This action involves no loss for him as Mike holds 100% of the shares. All subsequent depositors will have their DAI investments rounded to `10^6 * 1e18`, due to the lack of precision which initial tiny deposit caused, with the remainder divided between all current depositors, i.e. the subsequent depositors lose value to Mike, who is effectively stealing the remainder values of deposits.

For example, if the second depositor brings in `1.8 * 10^6 * 1e18`, only `1` of new LToken to be issued, which means that `2.8 * 10^6 * 1e18` total DAI pool be divided 50/50 between the second depositor and Mike, as each have `1 wei` of the total `2 wei` of LToken shares, i.e. the depositor lost and Mike gained `0.4 * 10^6 * 1e18` or `400k` DAI.

Mike can remain invested for an arbitrary time, gathering the share of all new deposits' remainder amounts.

## Impact

The principal funds loss for many users (most of depositors) is the impact here, however only new LToken and LEther contract instances are vulnerable, so placing severity to be **medium**.

All subsequent depositors' remainder amounts will be similarly divided between current depositors, providing marginally decreasing net gain for the attacker. Attacker's investment is also kept, and the whole attack is economically viable with a substantial return for him and thus probable.

## Code Snippet

LToken's and LEther's deposit obtain number of shares with previewDeposit():

LToken:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/utils/ERC4626.sol#L48-L60

LEther:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LEther.sol#L26-L38

previewDeposit() divides total supply by totalAssets():

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/utils/ERC4626.sol#L138-L140

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/utils/ERC4626.sol#L126-L130

LToken's totalAssets() (LEther inherits it) estimate is based on the current balance with an `asset`:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LToken.sol#L185-L193

Balance can be manipulated both in LToken and LEther cases (prime way is direct token or native funds transfer; other ways can be possible, but it's not significant here), which provides the attack surface described.

In the example outlined above `assets.mulDivDown(supply, totalAssets()) = 1.8 * 10^6 * 1e18 * 1 / 10^6*1e18 = 1` as `supply = 1 wei`, while `totalAssets() = asset.balanceOf(address(this)) = 10^6*1e18`.

As Mike also holds `1 wei` of shares, this involves that after the call completes he will have claim for the half of the `totalAssets() = asset.balanceOf(address(this)) = 2.8 * 10^6 * 1e18`.

## Recommendation

A minimum for the user provided deposit amount can drastically reduce the economic viability of the attack. I.e. ERC4626's deposit() and LEther's depositEth() can require each deposit to surpass the threshold, and then an attacker would have to provide too big direct investment to capture any meaningful share of the subsequent deposits.

An alternative is to require only the first depositor to freeze big enough initial amount of liquidity. This approach has been used long enough in Uniswap V2:

https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L121