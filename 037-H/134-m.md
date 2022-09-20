xiaoming90
# CTokenOracle Vulnerable To Price Manipulation

## Summary

CToken oracle is vulnerable to price manipulation. This allows an attacker to increase or decrease the price of the CToken and CEther to carry out various attacks against the protocol.

## Vulnerability Detail

The `getCEtherPrice` and `getCErc20Price` functions within the `CTokenOracle` contract are vulnerable to price manipulation because the price can be increased or decreased in the same transaction.

Based on the `getCEtherPrice` and `getCErc20Price` functions, the price of CEther and CErc tokens is dependent on the `exchangeRateStored` function. If the value return by the `exchangeRateStored` function can be manipulated within a single transaction/block, the price of CEther and CErc tokens is considered to be vulnerable to price manipulation.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/oracle/src/compound/CTokenOracle.sol#L54

```solidity
File: CTokenOracle.sol
54:     function getCEtherPrice() internal view returns (uint) {
55:         /*
56:             cToken Exchange rates are scaled by 10^(18 - 8 + underlying token decimals) which comes
57:             out to 28 decimals for cEther. We must divide the exchange rate by 1e10 to scale it to
58:             18 decimals. Finally we multiply this with the price of the underlying token, in this
59:             case the price of ETH - 1e18. In the implementation below we combine these two ops and
60:             thus the cEther price can be computed as --
61:             exchangeRateStored() / 1e10 * 1e18 = exchangeRateStored * 1e8
62:         */
63:         return ICToken(cETHER).exchangeRateStored().mulWadDown(1e8);
64:     }
65: 
66:     function getCErc20Price(ICToken cToken, address underlying) internal view returns (uint) {
67:         /*
68:             cToken Exchange rates are scaled by 10^(18 - 8 + underlying token decimals) so to scale
69:             the exchange rate to 18 decimals we must multiply it by 1e8 and then divide it by the
70:             number of decimals in the underlying token. Finally to find the price of the cToken we
71:             must multiply this value with the current price of the underlying token
72:         */
73:         return cToken.exchangeRateStored()
74:         .mulDivDown(1e8 , IERC20(underlying).decimals())
75:         .mulWadDown(oracle.getPrice(underlying));
76:     }
```

Per the implementation of Compound's `exchangeRateStored` function, the spot/current `totalSupply` is used to derive the exchange rate. The `totalSupply` can be increased or decreased within a single block/transaction by calling the cToken's mint or redeem function. This allows the attacker to artificially inflate or deflate the price within a single block/transaction.

https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CToken.sol#L284

```solidity
function exchangeRateStored() override public view returns (uint) {
    return exchangeRateStoredInternal();
}

/**
 * @notice Calculates the exchange rate from the underlying to the CToken
 * @dev This function does not accrue interest before calculating the exchange rate
 * @return calculated exchange rate scaled by 1e18
 */
function exchangeRateStoredInternal() virtual internal view returns (uint) {
    uint _totalSupply = totalSupply;
    if (_totalSupply == 0) {
        /*
         * If there are no tokens minted:
         *  exchangeRate = initialExchangeRate
         */
        return initialExchangeRateMantissa;
    } else {
        /*
         * Otherwise:
         *  exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
         */
        uint totalCash = getCashPrior();
        uint cashPlusBorrowsMinusReserves = totalCash + totalBorrows - totalReserves;
        uint exchangeRate = cashPlusBorrowsMinusReserves * expScale / _totalSupply;

        return exchangeRate;
    }
}
```

## Impact

The attacker could perform price manipulation to make the apparent value of an asset to be much higher or much lower than the true value of the asset. Following are some risks of price manipulation:

- An attacker can increase the value of their collaterals to increase their borrowing power so that they can borrow more assets than they are allowed from Sentiment.
- An attacker can decrease the value of some collaterals and attempt to liquidate another user account prematurely.

## Recommendation

Avoid using `exchangeRateStored` function to calculate the price of the CEther and CErc tokens. Consider implementing TWAP so that the price cannot be inflated or deflated within a single block/transaction or within a short period of time.