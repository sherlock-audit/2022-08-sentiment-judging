cergyk
# Oracle Consideration: Uniswap V2 price oracle can be manipulate by inflating the LP token balance

## Summary

Uniswap V2 price oracle can be manipulated by inflating the LP token balance using flashloan.

## Vulnerability Detail

In UniswapV2LPOracle.sol

the getPrice logic is implemented as

```
    function getPrice(address pair) external view returns (uint) {
        (uint r0, uint r1,) = IUniswapV2Pair(pair).getReserves();

        // 2 * sqrt(r0 * r1 * p0 * p1) / totalSupply
        return FixedPointMathLib.sqrt(
            r0
            .mulWadDown(r1)
            .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token0()))
            .mulWadDown(oracle.getPrice(IUniswapV2Pair(pair).token1()))
        )
        .mulDivDown(2e27, IUniswapV2Pair(pair).totalSupply());
    }
```

note the formula used is 

```
 2 * sqrt(r0 * r1 * p0 * p1) / totalSupply
```

but 

```
 IUniswapV2Pair(pair).totalSupply());
```

can be manipulated and inflated. If this number is inflated and oracle price is manipulated. Please check the POC below!

A malicious user can manipluated the oracle price in the following steps.

Suppose there a pool, with token0 as A, token0 as B.

```
(uint r0, uint r1,) = IUniswapV2Pair(pair).getReserves();
```

r0 is token0 -> A,
r1 is token1 -> B,

0. The user open a position.

1. The User flash loan a large amount of token A and token B.

2. The user adds loanded amount of token A and B into the liquidity pool of the token pair and mint a large amount of LP token to inflate the oracle price.

3. The user borrows a large excessive large number of tokens and never repays back and leaves the bad debt.

4. the user withdraws liquidity, repays the flash loan, and walks away with profit at the cost of the user. 

Please check the simulation code before in Python implementation, the price is manipulated from 20 to 500

```

import math

class Simulation:

    def __init__(self):
        self.reserve0 = 100
        self.reserve1 = 100
        self.oracle_for_reserve0 = 1
        self.oracle_for_reserve1 = 1
        self.total_lp_supply = 10

    def get_price(self):
        
        # 2 * sqrt(r0 * r1 * p0 * p1) / totalSupply

        return 2 * math.sqrt(
            self.reserve0 * self.oracle_for_reserve0 *
            self.reserve1 * self.oracle_for_reserve1
        ) / self.total_lp_supply

    # https://github.com/Uniswap/v1-periphery/blob/master/contracts/UniswapV1Router01.sol

    # https://www.reddit.com/r/UniSwap/comments/i49dmk/how_are_lp_token_amounts_calculated/

    # https://github.com/Uniswap/v2-core/blob/4dd59067c76dea4a0e8e4bfdda41877a6b16dedc/contracts/UniswapV2Pair.sol#L123

    # liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount0.mul(_totalSupply) / _reserve0);

    def inflate(self, amount0, amount1):
    
        self.reserve0 += amount0 # transfer amount0 into pair pool
        self.reserve1 += amount1 # transfer amount1 into pair pool.

        liquidity = min(
            amount0 * self.total_lp_supply / self.reserve0,
            amount1 * self.total_lp_supply / self.reserve1
        )

        self.total_lp_supply += liquidity # mint LP

if __name__ == "__main__":

    simulator = Simulation()

    price = simulator.get_price()
    print('price before', price)

    amount0 = 10000
    amount1 = 10000
    simulator.inflate(amount0, amount1)

    price = simulator.get_price()
    print('price after', price)
```

the running result is 

```
price before 20.0
price after 1015.0248756218905
```

## Impact

If the oracle data is invalid because of imbalanced token pool or very small price0 / price1, then the function _valueIn can get the invalid oracle and determine the wrong number of debt or callateral worth. User may not able to deposit or repay to manage their position or malicious liquidators can liquidated user's account balance or borrow an exessive amount and never repay the debt.

```
    function _valueInWei(address token, uint amt)
        internal
        view
        returns (uint)
    {
        return oracle.getPrice(token)
        .mulDivDown(
            amt,
            10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
        );
    }
```

## Code Snippet

## Tool used

Manual Review

## Recommendation

Use Strict oracle price instead of getting reserved amount and LP balance involed or use the TWAP oracle.
