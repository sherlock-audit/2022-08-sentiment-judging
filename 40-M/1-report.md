grhkm
# ERC20 tokens with multiple entrypoints can be doublecounted

## Summary
When an ERC20 token has multiple entrypoints, the balance will be counted two times.

## Vulnerability Detail
Some ERC20 tokens have multiple entrypoints, see e.g. here, how this affected Compound: https://medium.com/chainsecurity/trueusd-compound-vulnerability-bc5b696d29e2

In such cases, both addresses could potentially be added to the assets of an account, meaning that the balance of this token would be counted two times in `RiskEngine._getBalance`.
## Impact
When both entrypoints are whitelisted, the balance of this token will be counted two times.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/RiskEngine.sol#L157

## Tool used

Manual Review

## Recommendation
Never whitelist two entrypoints of the same ERC20 token and fix all issues (see other issues) where the added tokens are not validated / the whitelist can be circumvented.

**Note**: This is more an operational issue, so I could understand if it were out-of-scope. I just wanted to make sure that you are aware of this risk.