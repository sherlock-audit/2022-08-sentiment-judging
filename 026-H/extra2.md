hyh
# UniV2LPOracle's getPrice() is incorrect for pairs with underlying tokens whose decimals aren't 18


## Summary

UniV2LPOracle's `getPrice(pair)` has wrong magnitude for pairs whose underlying tokens' decimals aren't equal to 18 as the fair LP pricing formula used there operates token reserves and do not take into account token decimals which define the denomination of these reserves.

## Vulnerability Detail

UniV2LPOracle's fair LP token pricing approach do not take into account the decimals of Pair's underlying tokens, using their reserves figures as if their decimals are fixed to be 18. In fact reserves, being token amounts, have the decimals of the corresponding tokens, and these decimals can vary. Say USDC have decimals of 6, WBTC of 8, so is the corresponding reserves values.

In the same time in order for the net asset value computations to be correct Oracles' getPrice() should always have 18 decimals.

## Impact

If pair's underlying tokens have decimals different from 18 then UniV2LPOracle's getPrice() provides results of the incorrect magnitudes. This will have a range of impacts linked with underpricing or overpricing of the collateral and borrow assets, with the subsequent liquidations for the healthy positions on the one hand and the liquidations denial for the unhealthy ones along with the possibility of heavily undercollateralized borrowing on the another. The latter can be critical.

The price reported is positively correlated with the token decimals, so it is underpricing and the liquidations of healthy positions for the tokens with decimals lower than 18 (say USDC, USDT, WBTC), and overpricing and the prohibition of the liquidation of underwater positions for the tokens has decimals bigger than 18. In the latter case even small part of collateral being held in such LP tokens will prohibit the liquidation and allow for the highly undercollateralized borrowing (say borrowing 1000x of the collateral value).

Setting severity to be high as this can be straightforwardly exploited by an attacker with the monetary loss of the protocol up to the whole funds available for borrowing. Say once such LP is included by enabling the Oracle, the attacker immediately borrows all the funds available having the LP as the collateral, whose real market value is magnitudes smaller than one of the funds borrowed.

## Code Snippet

RiskEngine() treats `oracle.getPrice(token)` as having fixed decimals of 18:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/core/RiskEngine.sol#L178-L188

OracleFacade translates the price as it is:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/oracle/src/core/OracleFacade.sol#L39-L43

I.e. in order for asset value computations to be correct all the Oracles should ensure that resulting decimals of the getPrice() is exactly 18.

The UniV2LPOracle's `r0` and `r1` doesn't correct for `token0` and `token1` decimals, which affect the denomination of pair reserves:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/oracle/src/uniswap/UniV2LPOracle.sol#L37-L50

For example, when it's `(r0, r1) = (ETH, USDT)`, `r0` have decimals of 18, `r1` have decimals of 6, and UniV2LPOracle's getPrice() have decimals of `0.5 * (18 + 6 + 18 + 18 - (3 * 18)) + 27 - 18` = `12`.

If `r1` decimals be `36` (https://github.com/d-xo/weird-erc20#high-decimals), it would be `0.5 * (18 + 36 + 18 + 18 - (3 * 18)) + 27 - 18` = `27`.

mulWadDown()/mulDivDown():

https://github.com/transmissions11/solmate/blob/main/src/utils/FixedPointMathLib.sol#L12-L72

Uniswap V2 reserves are based on token balances and have the decimals of the corresponding tokens:

https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L38-L42

```
    function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
        _blockTimestampLast = blockTimestampLast;
    }
```

https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L109-L113

```
    // this low-level function should be called from a contract which performs important safety checks
    function mint(address to) external lock returns (uint liquidity) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
```

https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L128

```
_update(balance0, balance1, _reserve0, _reserve1);
```

https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L72-L83

```
    // update reserves and, on the first call per block, price accumulators
    function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
```

## Recommendation

Consider taking into account the decimals of the underlying tokes of a pair, for example:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/oracle/src/uniswap/UniV2LPOracle.sol#L37-L50

```solidity
    /// @dev Adapted from https://blog.alphaventuredao.io/fair-lp-token-pricing
    /// @inheritdoc IOracle
    function getPrice(address pair) external view returns (uint) {
+       address token0 = IUniswapV2Pair(pair).token0();
+       address token1 = IUniswapV2Pair(pair).token1();
+       uint decimals0 = token0.decimals();
+       uint decimals1 = token1.decimals();
        (uint r0, uint r1,) = IUniswapV2Pair(pair).getReserves();
+       r0 = decimals0 < 18 ? r0 * 10**(18 - decimals0) : r0 / 10**(decimals0 - 18);
+       r1 = decimals1 < 18 ? r1 * 10**(18 - decimals1) : r1 / 10**(decimals1 - 18);

        // 2 * sqrt(r0 * r1 * p0 * p1) / totalSupply
        return FixedPointMathLib.sqrt(
            r0
            .mulWadDown(r1)
-           .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token0()))
-           .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token1()))
+           .mulWadDown(oracle.getPrice(token0))
+           .mulWadDown(oracle.getPrice(token1))
        )
        .mulDivDown(2e27, IUniswapV2Pair(pair).totalSupply());
    }
```