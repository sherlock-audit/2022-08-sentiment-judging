xiaoming90
# ERC4626Oracle Vulnerable To Price Manipulation

## Summary

ERC4626 oracle is vulnerable to price manipulation. This allows an attacker to increase or decrease the price to carry out various attacks against the protocol.

## Vulnerability Detail

The `getPrice` function within the `ERC4626Oracle` contract is vulnerable to price manipulation because the price can be increased or decreased within a single transaction/block.

Based on the `getPrice` function, the price of the LP token of an ERC4626 vault is dependent on the `ERC4626.previewRedeem` and `oracleFacade.getPrice` functions. If the value returns by either `ERC4626.previewRedeem` or `oracleFacade.getPrice` can be manipulated within a single transaction/block, the price of the LP token of an ERC4626 vault is considered to be vulnerable to price manipulation.

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/oracle/src/erc4626/ERC4626Oracle.sol#L8

```solidity
File: ERC4626Oracle.sol
35:     function getPrice(address token) external view returns (uint) {
36:         uint decimals = IERC4626(token).decimals();
37:         return IERC4626(token).previewRedeem(
38:             10 ** decimals
39:         ).mulDivDown(
40:             oracleFacade.getPrice(IERC4626(token).asset()),
41:             10 ** decimals
42:         );
43:     }
```

It was observed that the `ERC4626.previewRedeem` couldbe manipulated within a single transaction/block. As shown below, the `previewRedeem` function will call the `convertToAssets` function. Within the `convertToAssets`, the number of assets per share is calculated based on the current/spot total assets and current/spot supply that can be increased or decreased within a single block/transaction by calling the vault's deposit, mint, withdraw or redeem functions. This allows the attacker to artificially inflate or deflate the price within a single block/transaction.

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/utils/ERC4626.sol#L154

```solidity
File: ERC4626.sol
154:     function previewRedeem(uint256 shares) public view virtual returns (uint256) {
155:         return convertToAssets(shares);
156:     }
```

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/tokens/utils/ERC4626.sol#L132

```solidity
File: ERC4626.sol
132:     function convertToAssets(uint256 shares) public view virtual returns (uint256) {
133:         uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.
134: 
135:         return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
136:     }
```

## Impact

The attacker could perform price manipulation to make the apparent value of an asset to be much higher or much lower than the true value of the asset. Following are some risks of price manipulation:

- An attacker can increase the value of their collaterals to increase their borrowing power so that they can borrow more assets than they are allowed from Sentiment.
- An attacker can decrease the value of some collaterals and attempt to liquidate another user account prematurely.

## Recommendation

Avoid using `previewRedeem` function to calculate the price of the LP token of an ERC4626 vault. Consider implementing TWAP so that the price cannot be inflated or deflated within a single block/transaction or within a short period of time.

## Sentiment Team
Acknowledged. Depends on the integration itself, so there's no action that can be taken right now.
