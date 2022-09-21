IllIllI
# Prices for ERC4626 tokens are not flash-loan-resistant

## Summary
The ERC4626 price oracle uses `previewRedeem()` to determine the price of the token.

## Vulnerability Detail
`previewRedeem()` includes fees, so it's not the actual price of the underlying assets. The EIP states that `previewRedeem()` `MUST be inclusive of deposit fees. Integrators should be aware of the existence of deposit fees`. Furthermore, the EIP states, `The preview methods return values that are as close as possible to exact as possible. For that reason, they are manipulable by altering the on-chain conditions and are not always safe to be used as price oracles. This specification includes convert methods that are allowed to be inexact and therefore can be implemented as robust price oracles. For example, it would be correct to implement the convert methods as using a time-weighted average price in converting between assets and shares.`
https://eips.ethereum.org/EIPS/eip-4626

## Impact
A flash loan may skew the price oracle, leading to the mispricing of assets, leading to loss of funds

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/oracle/src/erc4626/ERC4626Oracle.sol#L34-L43

## Tool used

Manual Review

## Recommendation
Use `convertToAssets()` instead
