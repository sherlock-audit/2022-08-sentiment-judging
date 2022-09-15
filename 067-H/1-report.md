csanuragjain
# User can borrow more than their balance

## Summary
Borrow of amount>userBalance is allowed via RiskEngine. 

## Vulnerability Detail

1. Assume User current account Balance is 10 and Borrow amount is 1

2. User account asks for borrow of amount 40. This makes call to [isBorrowAllowed](https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/RiskEngine.sol#L72) to verify if user account can be borrowed such amount

3. _isAccountHealthy is called internally by isBorrowAllowed as shown below

```
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

```
function _isAccountHealthy(uint accountBalance, uint accountBorrows)
        internal
        pure
        returns (bool)
    {
        return (accountBorrows == 0) ? true :
            (accountBalance.divWadDown(accountBorrows) > balanceToBorrowThreshold);
    }
```

5. So _isAccountHealthy is called with accountBalance as 10+40=50 and borrows as 1+40=41

6. This become equivalent to:

```
(accountBalance.divWadDown(accountBorrows) > balanceToBorrowThreshold) // (10+40)/(1+40) = 1.21 > 1.2
```

7. Hence the borrow is success even though amount is greater than user balance

## Impact
User can borrow more than collateral provided which could lead to huge financial losses to the sentiment protocol

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/RiskEngine.sol#L72

## Tool used
Manual Review

## Recommendation
Change _isAccountHealthy call in [isBorrowAllowed](https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/RiskEngine.sol#L72) function as described below:

```
return _isAccountHealthy(
            _getBalance(account),
            _getBorrows(account) + borrowValue
        );
```