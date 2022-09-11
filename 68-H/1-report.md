grhkm
# User funds can be lost

## Summary
User funds could be lost if tokens are mix of ERC20 token with different decimal places. 

## Vulnerability Detail
1. Account A has 3 assets with 10,18,24 decimal places and amount as 1 each
2. Now [_getBalance](https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/RiskEngine.sol#L150) is checked for this account which calls _valueInWei for each token

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

3. The calculation looks like (assuming all have price as 1)

```
1 * (1 * 10^10) = 10^10 
1 * (1 * 10^18) = 10^18 
1 * (1 * 10^24) = 10^24 
```

4. So totalBalance becomes

```
totalBalance = 10^10 + 10^18 + 10^24 = 10^10 (1 + 10^8 + 10^14) = 10^10 (1 + 10^8 ( 1 + 10^6)) = 100000100000001000000000
```

5. As we can see token with 10 decimal places loses its value highly due to precision and becomes negligible

## Impact
User funds could be lost 

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/RiskEngine.sol#L178

## Tool used
Manual Review

## Recommendation
Use standard 18 decimal places by either padding (lower than 18 decimal places) or killing (higher than 18 decimal places) zero