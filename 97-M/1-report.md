PwnPatrol
# Missing revert keyword

## Summary
Missing `revert` keyword makes edge case success silently.

## Vulnerability Detail
Helper function `functionDelegateCall` lacks `revert` keyword which should be thrown when the address is not a contract. If the target is EOA (or a contract without code) then the delegate would be called and it would be successful (because EVM treats no code as STOP opcode).

## Impact
Delegate calls would be successful (but they should fail) and possibly unnoticed.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-PwnPatrol0x/blob/main/protocol/src/utils/Helpers.sol#L66-L73
```solidity
    function functionDelegateCall(
        address target,
        bytes calldata data
    ) internal {
        if (!isContract(target)) Errors.AddressNotContract;
        (bool success, ) = target.delegatecall(data);
        require(success, "CALL_FAILED");
    }
```

## Tool used

Manual Review

## Recommendation
Add missing `revert` keyword