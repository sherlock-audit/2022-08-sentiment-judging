minhquanym
# `lTokens` list of Registry can contain duplicated elements and removed LToken

## Vulnerability Detail

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L95

In `Registry` contract, there is a `lTokens` which keep the list of active LToken. Only admin can add/remove/update elements of this list. The problem is, in `setLToken()` function, it didn't check if new lToken is already existed in `lTokens` list or not.
```solidity
function setLToken(address underlying, address lToken) external adminOnly {
    if (LTokenFor[underlying] == address(0)) {
        if (lToken == address(0)) revert Errors.ZeroAddress();
        lTokens.push(lToken);
    }
    else if (lToken == address(0)) removeLToken(LTokenFor[underlying]);
    else updateLToken(LTokenFor[underlying], lToken);

    LTokenFor[underlying] = lToken;
}
```

And in case `lTokens` list has duplicated element, after admin try to remove a LToken, `getAllLTokens()` still return a list included removed one because `removeLToken()` just deleted the first element it found, left duplicated one untouched.

## Impact

Calling `Registry.getAllLTokens()` might return duplicated elements and inactive one. 
But since it can only be manipulated by admin, the impact is reduced significantly.

## Proof of Concept

Consider the scenario
1. Admin called `setLToken()` 2 times with the same `lToken` param and different `underlying`. For example
```solidity
setLToken(USDT, lUSD);
setLToken(USDC, lUSD);
```
Since there is no check, both calls are not reverted, results in `lTokens = [lUSD, lUSD]` which is duplicated.

2. Now admin found out and want to deactivate `lUSD` by calling
```solidity
setLToken(USDC, lUSD);
```
But `lUSD` is only removed once from `lTokens` list, result in `lTokens = [lUSD]` which contained inactive LToken.

## Tool used

Manual Review

## Recommendation

Check if `LToken` existed first before add into `lTokens` list