grhkm
#  Low-level transfer via call() can fail silently

##  Low-level transfer via call() can fail silently

**Context:**
[Account.sol#L154](https://github.com/sherlock-audit/2022-08-sentiment-0xSmartContract/blob/main/protocol/src/core/Account.sol#L154)

**Description:**
Solidity docs:
"The low-level functions call, delegatecall and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed."
 Therefore, transfers may fail silently.


in the Account.sol a call is executed with the following code in ```exec``` function

```js
function exec(address target, uint amt, bytes calldata data)
        external
        accountManagerOnly
        returns (bool, bytes memory)
    {
        (bool success, bytes memory retData) = target.call{value: amt}(data);
        return (success, retData);
    }
```

**Proof of Concept**
Please find the documentation here: [https://docs.soliditylang.org/en/develop/control-structures.html#error-handling-assert-require-revert-and-exceptions](https://docs.soliditylang.org/en/develop/control-structures.html)


**Recommendation:**
Check for the account's existence prior to low level call

call is not recommended in most situations for contract function calls because it bypasses type checking, function existence check, and argument packing. It is preferred to import the interface of the contract to call functions on it.