WATCHPUG
# `UniV2LPOracle` will malfunction if token0 or token1's `decimals != 18`

## Summary

When one of the LP token's underlying tokens `decimals` is not 18, the price of the LP token calculated by `UniV2LPOracle` will be wrong. 

## Vulnerability Detail

`UniV2LPOracle` is an implementation of Alpha Homora v2's Fair Uniswap's LP Token Pricing Formula:

> The Formula ... of combining fair asset prices and fair asset reserves:

> $$
P = 2\cdot \frac{\sqrt{r_0 \cdot r_1} \cdot \sqrt{p_0\cdot p_1}}{totalSupply},
$$

> where $r_i$ is the asset ii's pool balance and $p_i$ is the asset $i$'s fair price.

However, the current implementation wrongful assumes $r_0$ and $r_1$ are always in 18 decimals.

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/uniswap/UniV2LPOracle.sol#L39-L50

```solidity
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

https://github.com/transmissions11/solmate/blob/main/src/utils/FixedPointMathLib.sol

```solidity
uint256 internal constant WAD = 1e18; // The scalar of ETH and most ERC20s.

function mulWadDown(uint256 x, uint256 y) internal pure returns (uint256) {
    return mulDivDown(x, y, WAD); // Equivalent to (x * y) / WAD rounded down.
}
```

https://github.com/transmissions11/solmate/blob/main/src/utils/FixedPointMathLib.sol

```solidity
function mulDivDown(
    uint256 x,
    uint256 y,
    uint256 denominator
) internal pure returns (uint256 z) {
    assembly {
        // Store x * y in z for now.
        z := mul(x, y)

        // Equivalent to require(denominator != 0 && (x == 0 || (x * y) / x == y))
        if iszero(and(iszero(iszero(denominator)), or(iszero(x), eq(div(z, x), y)))) {
            revert(0, 0)
        }

        // Divide z by the denominator.
        z := div(z, denominator)
    }
}
```

## Impact

When the decimals of one or both tokens in the pair is not 18, the price will be way off.

## Code Snippet

We've created a test script to demonstrate `UniV2LPOracle` is malfunctioning with USDC/WETH, in which USDC's decimals is 6 instead of 18.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "forge-std/console.sol";
import {Test} from "forge-std/Test.sol";

import {IOracle} from "../core/IOracle.sol";
import {UniV2LpOracle} from "../uniswap/UniV2LPOracle.sol";

import {OracleFacade} from "../core/OracleFacade.sol";
import {ChainlinkOracle} from "../chainlink/ChainlinkOracle.sol";
import {WETHOracle} from "../weth/WETHOracle.sol";
import {AggregatorV3Interface} from "../chainlink/AggregatorV3Interface.sol";
import "forge-std/console.sol";

contract UniV2LPOracleTest is Test {
    address constant usdc = address(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    address constant weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    address constant pair = address(0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc);

    address constant ethUsdFeed =
        address(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
    address constant usdcUsdFeed =
        address(0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6);

    ChainlinkOracle chainlinkOracle;
    WETHOracle wETHOracle;
    UniV2LpOracle uniV2LpOracle;
    OracleFacade oracleFacade;

    function setUp() public {
        // core oracle
        oracleFacade = new OracleFacade();

        chainlinkOracle = new ChainlinkOracle(
            AggregatorV3Interface(ethUsdFeed)
        );
        chainlinkOracle.setFeed(usdc, AggregatorV3Interface(usdcUsdFeed));

        WETHOracle wethOracle = new WETHOracle();
        oracleFacade.setOracle(weth, wethOracle);
        oracleFacade.setOracle(usdc, chainlinkOracle);

        uniV2LpOracle = new UniV2LpOracle(oracleFacade);
    }

    function testUniV2Price() public {
        console.log(uniV2LpOracle.getPrice(pair));
    }
}

```

#### `reserves` and `totalSupply` of UniswapV2 USDC/ETH Pair `0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc` on Mainnet at the time of writing

Result of `getReserves()`
```js
_reserve0   uint112 :  45456843739761
_reserve1   uint112 :  28342500764440756425363
_blockTimestampLast   uint32 :  1663233599
```

`totalSupply()`: `550760227054391377`

#### Expected and actual result

```js
// real time price
28342500764440756425363 * 2   / 550760227054391377 = 102921

// normalized r0 and r1 ✅
102728696484607347546879 / 1e18 = 102728

// current result
102728772378134052 / 1e18 = 0.10272877237813405
```

## Tool used

Manual Review

## Recommendation

Consider normalizing r0 and r1 to 18 decimals before using them in the formula.