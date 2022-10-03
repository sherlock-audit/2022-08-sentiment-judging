Chom
# User can block liquidation by borrowing too many assets

## Summary
Users can block liquidation by borrowing too many assets

## Vulnerability Detail

```solidity
        for(uint i; i < borrowLen; ++i) {
            address token = accountBorrows[i];
            LToken = ILToken(registry.LTokenFor(token));
```

_liquidate is looping over accountBorrows, if a user is borrowing too many assets for example 100 assets, accountBorrows will have a length of 100. Looping over 100 elements while performing complex operations may cause the gas limit revert.

## Impact
If a user is borrowing too many assets for example 100+ assets, nobody will be able to liquidate that user, causing serious bad debt. Attackers can use this vulnerability to borrow assets risk-free. 

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L250-L255

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L367-L385

## Tool used

Manual Review

## Recommendation

Liquidation and borrowing should be done per asset instead of a global pooling of all assets.
