cryptphi
# Missing access control can allow anyone to initialize Registry and become owner

## Summary
Registry.init() is missing an access control which allows anyone to call the init() function and become the owner of the contract. This also allows for potential frontrunning of the function allowing an attacker to frontrun the transaction to initialize the contract and become the owner.

## Vulnerability Detail
Registry.init() initializes the contract and sets the caller to become the owner of the contract allowing the caller to perform any admin privilege activity. Since there is no access control check, anyone can call the function and become the owner if has not been initialized from on set.
Additionally, if an authorized address calls the init() function, due to not having access control, any user monitoring the blockchain network will replay the call and frontrun their transaction to be the one added to the block successfully and be the owner of the Registry contract.

## Impact
Access Control issues, arbitrary user can control the  contracts setLToken() and setAddress() functions

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L61-L65

```
function init() external {
        if (initialized) revert Errors.ContractAlreadyInitialized();
        initialized = true;
        initOwnable(msg.sender);
    }
```

## Tool used

Manual Review

## Recommendation
Add a check to ensure that the caller is an authorized address.
