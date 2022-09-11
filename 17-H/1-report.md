grhkm
# ChainlinkOracle: Age of response not validated

## Summary
The responses of chainlink oracles are allowed to be arbitrarily old, which can be exploited to drain the protocol.

## Vulnerability Detail
`ChainlinkOracle` uses the latest value of `latestRoundData()`, no matter how old it is. 

## Impact
If a chainlink feed was not updated in a long time and the price dropped, an attacker can buy a lot of these tokens and deposit them as collateral. The system will still use the old price, but the tokens may be almost valueless (e.g., UST) now. Because of this, the attacker can take out loans even if his health factor (with the real price) would be significantly below 1.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/oracle/src/chainlink/ChainlinkOracle.sol#L67

## Tool used

Manual Review

## Recommendation
Validate `updatedAt` of `latestRoundData()` and check that the price age is not above a threshold. If it is, do not allow operations with this token.