cergyk
# CTokenOracle: Missing normalization by 10^18

## Summary
The prices that are returned by `getCErc20Price` need to be normalized.

## Vulnerability Detail
In `getCErc20Price`, there is a multiplication of the CErc20/Erc20 price by the Erc20/ETH price in the end. However, because the Erc20/ETH price is returned with 18 decimals, a division by 1e18 is needed in the end.

## Impact
The prices are currently wrong.
Let's take cUSDC (https://etherscan.io/token/0x39aa39c021dfbae8fac545936693ac917d5e7563#readContract) with a current price of ~0.000014 ETH. Currently, `exchangeRateStored()` is 226405222044735 and `getCErc20Price` would return:
```
((226405222044735 * 1e8) / 1e6) * oracle.getPrice(address(USDC) = 
((226405222044735 * 1e8) / 1e6) * 626633162680235 = 
1.41e31 
```

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/compound/CTokenOracle.sol#L73

## Tool used

Manual Review

## Recommendation
```
        return cToken.exchangeRateStored()
        .mulDivDown(1e8 , IERC20(underlying).decimals())
        .mulWadDown(oracle.getPrice(underlying))
        .divWadDown(1e18);
```
Then, the above example returns 1.41e13, which is correct.