kirk-baird
# MED: `Registry` Contains Numerous Unbounded Loops

## Summary

`Registry.accountsOwnedBy()` is an unbounded loop on the total number of created accounts. This will break the block gas limit and revert after a few hundred accounts are created. Furthermore, as more user create accounts this will eventually cause RPC requests to `eth_call` to timeout. 

Futhermore, the following functions are unbounded in either loop iterations or return size and will eventually breach block gas limits or the RPC limits.
- `getAllKeys()`
- `getAllAccounts()`
- `getAllLTokens()`
- `updateLToken()`
- `removeLToken()`
- `removeKey()`

## Vulnerability Detail

Iterating through an unbounded loop will eventually consume as much gas as in the block gas limiit. Even if the gas limit is removed for an RPC call it will timeout when too many accounts are created.

## Impact

The impact is that these functions will not be able to be called on-chain or off-chain.

Iterating over `accounts` is the most dangerous as this length is the number of user accounts which can be trivially created by an attacker (or genuine user). The other fields are controlled by the admin and so are lower risk.

## Code Snippet

```solidity
    function accountsOwnedBy(address user)
        external
        view
        returns (address[] memory userAccounts)
    {
        userAccounts = new address[](accounts.length);
        uint index;
        for (uint i; i < accounts.length; i++) {
            if (ownerFor[accounts[i]] == user) {
                userAccounts[index] = accounts[i];
                index++;
            }
        }
        assembly { mstore(userAccounts, index) }
    }
```

```solidity
    function updateLToken(address lToken, address newLToken) internal {
        uint len = lTokens.length;
        for(uint i; i < len; ++i) {
            if(lTokens[i] == lToken) {
                lTokens[i] = newLToken;
                break;
            }
        }
    }

    function removeLToken(address underlying) internal {
        uint len = lTokens.length;
        for(uint i; i < len; ++i) {
            if (underlying == lTokens[i]) {
                lTokens[i] = lTokens[len - 1];
                lTokens.pop();
                break;
            }
        }
    }

    function removeKey(string calldata id) internal {
        uint len = keys.length;
        bytes32 keyHash = keccak256(abi.encodePacked(id));
        for(uint i; i < len; ++i) {
            if (keyHash == keccak256(abi.encodePacked((keys[i])))) {
                keys[i] = keys[len - 1];
                keys.pop();
                break;
            }
        }
    }
```
## Tool used

Manual Review

## Recommendation

Consider using mappings instead of arrays for the data structures. This will prevent retrieving all items in the array but will greatly improve updates, deletions and checks for existence gas usage and will remove the size restrictions.
