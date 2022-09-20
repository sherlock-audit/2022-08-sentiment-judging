Waze
# Failed transfer with low level call would not revert

## Summary

According to the Solidity docs), "The low-level functions call, delegatecall and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed".

## Vulnerability Detail

The exec() will not notice anything went wrong. This may result in user funds lost because funds were transferred. The exec fails but doesn't revert.

## Impact
When the contract's existence is not verified (target), the call will fails but returns success due to nonexistance contract and we receive nothing on return.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-calmkidd/blob/5d8b0126f4e4dcdab99c5c89922d74faa99cecb3/protocol/src/core/Account.sol#L149-L155

## Tool used

Manual Review

## Recommendation

Check for contract existence on low-level calls, so that failures are not missed.

A similar issue was awarded a medium here :
https://github.com/code-423n4/2022-01-trader-joe-findings/issues/170