WATCHPUG
# UniswapV3 TWAP Oracle without minimal liquidity check can be manipulable

## Summary

Unlike Uniswap v2, the special concentrated liquidity design of Uniswap V3 makes it prone to price manipulation, especially for a pair with rather low liquidity.

## Vulnerability Detail

UniswapV3 TWAP Oracle utilizes the spot price on Uniswap v3 for an asset over a specified period in the past; which returns a time-weighted average price (TWAP) for an asset, rather than the current spot price.

These oracles may be vulnerable to manipulation when the Uniswap V3 pool they derive from is illiquid or thinly traded.

Euler Finance has done some in-depth research on this: https://www.euler.finance/blog/euler-protocols-oracle-risk-grading-system

There are already some past events exploited with oracle manipulation due to the lack of liquidity in the Uniswap V3 pair. E.g: https://twitter.com/FloatProtocol/status/1482184042850263042

See also: https://www.euler.finance/blog/uniswap-oracle-attack-simulator

## Impact

If a pair with low liquidity is added or an existing pair's liquidity drops to a certain level, the attacker can inflate the price and borrow and withdraw valuable tokens, leading to bad debt to the protocol.

## Code Snippet

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/uniswap/UniV3TWAPOracle.sol#L41-L59

```solidity
    function getPrice(address token) public view returns (uint256) {

        address pool;
        if ((pool = poolFor[token]) == address(0)) {
            revert Errors.PriceUnavailable();
        }

        (int24 arithmeticMeanTick, ) = OracleLibrary.consult(
            pool,
            twapPeriod
        );

        return OracleLibrary.getQuoteAtTick(
            arithmeticMeanTick,
            uint128(10) ** IERC20(token).decimals(),
            token,
            WETH
        );
    }
```

## Tool used

Manual Review

## Recommendation

1. Consider adding a minimum liquidity check and only allows adding a pair with, say 1000 ETH of liquidity.
3. Consider adopting Euler Protocol’s `Oracle Risk Grading System`

https://www.euler.finance/blog/euler-protocols-oracle-risk-grading-system