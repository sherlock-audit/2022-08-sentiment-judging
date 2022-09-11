grhkm
# Failed transfer with low level call won't revert

## Summary

https://github.com/sherlock-audit/2022-08-sentiment-andyfeili/blob/96338b720493bc6dcbfa8ed24b75af53adc7900d/protocol/src/core/Account.sol#L149-L156

## Vulnerability Detail

An account can call an arbitrary contract using the `exec()` function. The function should return success True if transaction was successful, false otherwise. 

However low level calls (call, delegate call and static call) return success if the called contract doesn’t exist (not deployed or destructed). This could result in the call failing, however success will be set to true, which means the call will not revert but fail silently. 

https://github.com/sherlock-audit/2022-08-sentiment-andyfeili/blob/96338b720493bc6dcbfa8ed24b75af53adc7900d/protocol/src/core/AccountManager.sol#L306-L308

Reference: [Solidity Docs](https://docs.soliditylang.org/en/develop/control-structures.html#error-handling-assert-require-revert-and-exceptions)

## Recommendation

Add validation to target address

```solidity
require(0 != address(target).code.length)
```
