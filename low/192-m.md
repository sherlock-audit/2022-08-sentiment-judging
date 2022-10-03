Chom
# Add assets without depositing anything

## Summary
Add assets without depositing anything

## Vulnerability Detail
```
    function deposit(address account, address token, uint amt)
        external
        whenNotPaused
        onlyOwner(account)
    {
        if (!isCollateralAllowed[token])
            revert Errors.CollateralTypeRestricted();
        if (IAccount(account).hasAsset(token) == false)
            IAccount(account).addAsset(token);
        token.safeTransferFrom(msg.sender, account, amt);
    }
```

Users can call deposit(account, token, 0) to add that token to the assets list via this code block.

```
        if (IAccount(account).hasAsset(token) == false)
            IAccount(account).addAsset(token);
```

But since amt is 0, the user doesn't actually transfer any token to the account contract.

## Impact
The account contract will have assets without any balance.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/AccountManager.sol#L162-L172

## Tool used

Manual Review

## Recommendation
Check amt > 0

```
    function deposit(address account, address token, uint amt)
        external
        whenNotPaused
        onlyOwner(account)
    {
        if (!isCollateralAllowed[token])
            revert Errors.CollateralTypeRestricted();
        if (amt == 0)
            revert Errors.ZeroAmount();
        if (IAccount(account).hasAsset(token) == false)
            IAccount(account).addAsset(token);
        token.safeTransferFrom(msg.sender, account, amt);
    }
```