minera
# ERC4626 Oracle may return incorrect price

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/erc4626/ERC4626Oracle.sol#L35-L43
## [M] ERC4626 Oracle may return incorrect price
### Problem
`decimals()` of underlying asset of ERC4626 and vault's `decimals()` may have different values
as result price calculated incorrectly.
### Proof of Concept
```solidity
oracle/src/erc4626/ERC4626Oracle.sol
35:     function getPrice(address token) external view returns (uint) {
36:         uint decimals = IERC4626(token).decimals();
37:         return IERC4626(token).previewRedeem(
38:             10 ** decimals
39:         ).mulDivDown(
40:             oracleFacade.getPrice(IERC4626(token).asset()),
41:             10 ** decimals  //@audit decimals of asset may be not equal to vault decimals
42:         );
43:     }
```
we get price Per Share `previewRedeem(10**shareTokenDecimals)`
and then calcultaing price `pricePerShare * assetPriceInETH / 10**shareTokenDecimals`
if `shareTokenDecimals` is not equal to asset decimals then we get something not useful
### Mitigation
Denominator should be underlying asset decimals
```diff
function getPrice(address token) external view returns (uint) {
        uint decimals = IERC4626(token).decimals();
+	uint assetDecimals = (IERC4626(token).asset()).decimals();
        return IERC4626(token).previewRedeem(
            10 ** decimals
        ).mulDivDown(
            oracleFacade.getPrice(IERC4626(token).asset()),
-            10 ** decimals
+	     10 ** assetDecimals
        );
    }
```
Reminder how you did it correctly in YTokenOracle:
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/yearn/YTokenOracle.sol#L38-L43
