IllIllI
# Prices using wrong decimals returned for some Chainlink oracles

## Summary
Chainlink uses 8 decimals for stablecoin-denominated prices, but 18 decimals for crypto-denominated ones

## Vulnerability Detail
The Chainlink oracle code assumes USDC prices, but the oracle may fetch an Eth price, and the decimals will be 18, rather than the expected 8, and so the division by the USDC-denominated Eth price fetched by getEthPrice() won't properly cancel the decimal units

## Impact
Mispriced assets leading to loss funds of the protocol

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/2e25699040ed87a9af62f2b637eafcc6b0b59cc5/oracle/src/chainlink/ChainlinkOracle.sol#L48-L59

https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/2e25699040ed87a9af62f2b637eafcc6b0b59cc5/oracle/src/core/IOracle.sol#L6-L11

## Tool used

Manual Review

## Recommendation
Check the number of decimals, and use the correct divisor
