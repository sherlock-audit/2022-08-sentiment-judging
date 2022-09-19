GimelSec
# Some tokens may only approve non-zero allowances once

## Summary

AccountManager gives a spender approval to spend a given amount of token from the account. But some tokens like USDT only `approve` non-zero allowances once.

## Vulnerability Detail

Some token like USDT only `approve` non-zero allowances once: `require(!((_value != 0) && (allowed[msg.sender][_spender] != 0)));`

AccountManager just calls `approve` to set allowances, which will be reverted if users want to change allowances.

## Impact

The `approve` function will not work, and users could not change the allowances.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-rayn731/blob/main/protocol/src/core/AccountManager.sol#L277

## Tool used

Manual Review

## Recommendation

Approve 0 before approving actual allowance value.
