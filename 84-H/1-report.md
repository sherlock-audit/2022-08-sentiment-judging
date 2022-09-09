minera
# `exec` function should check if the `target` is a contract

##  ```exec``` function should check if the ```target``` is a contract

**Context:**
[Account.sol#L154](https://github.com/sherlock-audit/2022-08-sentiment-0xSmartContract/blob/main/protocol/src/core/Account.sol#L154)

**Description:**
```exec``` function not payable so we believe it's not the desired behavior to call a non-contract address and consider it a successful call.

For example, if a certain business logic requires a successful token.transferFrom() call to be made with the Account.sol, if the token is not a existing contract, the call will return success: true instead of success: false and break the caller's assumption and potentially malfunction features or even cause fund loss to users.


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
The qBridge exploit (January 2022) was caused by a similar issue.
https://halborn.com/explained-the-qubit-hack-january-2022/




**Recommendation:**

As a reference, OpenZeppelin's Address.functionCall() will check and require(isContract(target), "Address: call to non-contract");

```js

 function isContract(address account) internal view returns (bool) {
        // This method relies on extcodesize/address.code.length, which returns 0
        // for contracts in construction, since the code is only stored at the end
        // of the constructor execution.

        return account.code.length > 0;
    }



 function functionCallWithValue(
        address target,
        bytes memory data,
        uint256 value,
        string memory errorMessage
    ) internal returns (bytes memory) {
        require(address(this).balance >= value, "Address: insufficient balance for call");
        require(isContract(target), "Address: call to non-contract");

        (bool success, bytes memory returndata) = target.call{value: value}(data);
        return verifyCallResult(success, returndata, errorMessage);
    }



```
Consider adding a check and throw when the target is not a contract.


```exec``` function  works correctly as in the line below:

```js
import "@openzeppelin/contracts/utils/Address.sol";

    using Address for address;

 function exec(address target, uint amt, bytes calldata data)
        external
        accountManagerOnly
        returns (bool, bytes memory)
    {
        require(target.isContract(), "Account: target is not a contract");
        (bool success, bytes memory retData) = target.call{value: amt}(data);
        return (success, retData);
    }


```
