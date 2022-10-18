WATCHPUG

# `originationFee` may result in the borrower account becoming liquidatable immediately

## Summary

Note: This issue is a part of the extra scope added by Sentiment AFTER the audit contest. This scope was only reviewed by WatchPug and relates to these three PRs:

1. [Lending deposit cap](https://github.com/sentimentxyz/protocol/pull/234)
2. [Fee accrual modification](https://github.com/sentimentxyz/protocol/pull/233)
3. [CRV staking](https://github.com/sentimentxyz/controller/pull/41)

`originationFee` may result in the borrower account becoming liquidatable immediately.

## Vulnerability Detail

When checking `riskEngine.isBorrowAllowed()`, the `originationFee` of the borrow is not considered. Thus, when `originationFee` is large enough, the borrower account becomes liquidatable immediately.

For example:

Let's say USDC's `originationFee` is 30%;

Alice has `100 USDC`, and 0 debt in her account;
Alice borrowed `400 USDC`, received only `280 USDC` after the `originationFee`;
Alice's account is now liquidatable.
Actually, in the case above, Alice's account won't even get liquidated as all the assets are worth (380 USDC) less than the total debt (`400 USDC`).

## Impact

`originationFee` may result in the borrower account becoming liquidatable immediately.

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/f5a9089e87752986af522cc952f95beb037491c8/src/tokens/LToken.sol#L243-L245

```solidity
function updateOriginationFee(uint _originationFee) external adminOnly {
    originationFee = _originationFee;
}
```

https://github.com/sentimentxyz/protocol/blob/f5a9089e87752986af522cc952f95beb037491c8/src/core/AccountManager.sol#L203-L217

```solidity
function borrow(address account, address token, uint amt)
    external
    whenNotPaused
    onlyOwner(account)
{
    if (registry.LTokenFor(token) == address(0))
        revert Errors.LTokenUnavailable();
    if (!riskEngine.isBorrowAllowed(account, token, amt))
        revert Errors.RiskThresholdBreached();
    if (IAccount(account).hasAsset(token) == false)
        IAccount(account).addAsset(token);
    if (ILToken(registry.LTokenFor(token)).lendTo(account, amt))
        IAccount(account).addBorrow(token);
    emit Borrow(account, msg.sender, token, amt);
}
```

https://github.com/sentimentxyz/protocol/blob/f5a9089e87752986af522cc952f95beb037491c8/src/core/RiskEngine.sol#L72-L86

```solidity
function isBorrowAllowed(
    address account,
    address token,
    uint amt
)
    external
    view
    returns (bool)
{
    uint borrowValue = _valueInWei(token, amt);
    return _isAccountHealthy(
        _getBalance(account) + borrowValue,
        _getBorrows(account) + borrowValue
    );
}
```

## Tool used

Manual Review

## Recommendation

Consider asserting riskEngine.isAccountHealthy() after the borrow:

```solidity
function borrow(address account, address token, uint amt)
    external
    whenNotPaused
    onlyOwner(account)
{
    if (registry.LTokenFor(token) == address(0))
        revert Errors.LTokenUnavailable();
    if (IAccount(account).hasAsset(token) == false)
        IAccount(account).addAsset(token);
    if (ILToken(registry.LTokenFor(token)).lendTo(account, amt))
        IAccount(account).addBorrow(token);
    if (!riskEngine.isAccountHealthy(account))
        revert Errors.RiskThresholdBreached();
    emit Borrow(account, msg.sender, token, amt);
}
```

## Sentiment Team
Fixed as recommended. PR [here](https://github.com/sentimentxyz/protocol/pull/236).

## Lead Senior Watson
`riskEngine.isBorrowAllowed` should be removed as it's no longer needed.

## Sentiment Team
Pushed a commit to remove the redundant call to riskEngine, you can find it [here](https://github.com/sentimentxyz/protocol/pull/236/commits/bfc445b02784f8130181641ce0054382b4cc3ec5).

