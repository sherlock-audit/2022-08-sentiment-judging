hyh
# CTokenOracle's getPrice uses stored cToken price instead of the current

## Summary

CTokenOracle, that reports Compound tokens price, uses the cToken's exchangeRateStored(), which can yield stale results as it doesn't accrue loan interest. As interest piles up over time, such stale cToken value will be less than the current market one, so this can lead to the liquidations of the healthy positions.

## Vulnerability Detail

cToken's exchangeRateStored() does not accrue interest, so the exchange rate returned uses the previously recorded value, i.e. exchangeRateStored() data is frequently out-of-date. The scale of this can vary and can be bigger for the cTokens that don't have enough activity. While exchangeRateStored() consumes less gas, only exchangeRateCurrent() gives correct cToken pricing.

For an Account with heavy usage of the less liquid cTokens and overall borderline risk health conditions this can be a missing bit of valuation that enables the erroneous liquidation of the otherwise healthy position.

## Impact

The impact of the incorrect valuation can range up to the liquidation of the healthy Accounts, which is the permanent loss of the principal funds for their owners.

## Code Snippet

cToken's exchangeRateStored() is being used instead of exchangeRateCurrent():

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/oracle/src/compound/CTokenOracle.sol#L54-L76

Compound points to the exchangeRateCurrent() as the source of pricing data:

https://docs.compound.finance/v2/ctokens/#exchange-rate

The only difference is that exchangeRateStored() doesn't take into account up-to-date interest:

https://github.com/compound-finance/compound-protocol/blob/master/contracts/CToken.sol#L270-L286

```
    /**
     * @notice Accrue interest then return the up-to-date exchange rate
     * @return Calculated exchange rate scaled by 1e18
     */
    function exchangeRateCurrent() override public nonReentrant returns (uint) {
        accrueInterest();
        return exchangeRateStored();
    }

    /**
     * @notice Calculates the exchange rate from the underlying to the CToken
     * @dev This function does not accrue interest before calculating the exchange rate
     * @return Calculated exchange rate scaled by 1e18
     */
    function exchangeRateStored() override public view returns (uint) {
        return exchangeRateStoredInternal();
    }
```

Permanent usage of exchangeRateStored() is dangerous as it tends to become more outdated over time.

## Recommendation

As a direct (although costly) mitigation consider using exchangeRateCurrent() all the time.

As a more sophisticated take consider controlling the degree of staleness as it can build over time. For example, introduce the timestamp of the last exchangeRateCurrent() call and use exchangeRateStored() for the requests that happen to be within the threshold of this timestamp, calling exchangeRateCurrent() otherwise.

This way, if the activity isn't substantial, the proper update is needed and exchangeRateCurrent() be used. If the activity rises, there is no need to update every time and it's enough to do it with desired threshold controlled frequency.