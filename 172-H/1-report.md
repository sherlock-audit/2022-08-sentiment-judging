hansfriese
# Borrowers might repay with a different token.

## Summary
Borrowers might repay with a different token.


## Vulnerability Detail
https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L103

https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L128

https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L227


## Impact
When the admin registers LToken using [setLToken()](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95), it doesn't check the `underlying` is the same as the `asset` of LToken.

When users borrow the token and repay, there is no guarantee these two tokens are the same and will produce unexpected problems when they are different each other.


## Proof of Concept
- When users borrow tokens [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L214), it transfers the `asset` of the LToken [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L143).

- But when users repay, it transfers the registered `token` (not the inside asset of LToken) [here](https://github.com/sentimentxyz/protocol/tree/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L236).

So there would be a problem when `token != asset`.


## Tool used
Manual Review


## Recommendation
We should check if the `underlying` is the same as the `asset` of the LToken in `Registry.setLToken()`.

(Pseudocode)

```
function setLToken(address underlying, address lToken) external adminOnly {
    if (LTokenFor[underlying] == address(0)) {
        if (lToken == address(0)) revert Errors.ZeroAddress();
        lTokens.push(lToken);
    }
    else if (lToken == address(0)) removeLToken(LTokenFor[underlying]);
    else updateLToken(LTokenFor[underlying], lToken);

    if(lToken != address(0) && underlying != lToken.asset) //++++++++++++++++++++
        revert Error.DifferentToken; 

    LTokenFor[underlying] = lToken;
}
```