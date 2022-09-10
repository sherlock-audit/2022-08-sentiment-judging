cergyk
# any amount of withdraw limit in AccountManager.sol is allowed if the account has no debt.

## Summary

any amount of withdraw limit in AccountManager.sol is allowed if the account has no debt.

## Vulnerability Detail

when we wtithdraw fund, we call the function in the accountManager.sol

```
    function isWithdrawAllowed(
        address account,
        address token,
        uint amt
    )
        external
        view
        returns (bool)
    {
        if (IAccount(account).hasNoDebt()) return true;
        return _isAccountHealthy(
            _getBalance(account) - _valueInWei(token, amt),
            _getBorrows(account)
        );
    }
```

note if the account has no debt, the isWithdrawAllowed is always return true.

```
   function hasNoDebt() external view returns (bool) {
        return borrows.length == 0;
    }
```

so suppose the account owner deposit 1 ETH, 

and we call 

```
 riskEngine.isWithdrawAllowed(account, address(0), 10)
```

the user certainly cannot withdraw 10 ETH, but the function would return True if it has no debt.

## Tool used

Foundry

Manual Review

## Recommendation

We command check account balance in the funtion isWithdrawAllowed

```
    function isWithdrawAllowed(
        address account,
        address token,
        uint amt
    )
        external
        view
        returns (bool)
    {   
        if(amt > account.balance) return false;
        if (IAccount(account).hasNoDebt()) return true;
        return _isAccountHealthy(
            _getBalance(account) - _valueInWei(token, amt),
            _getBorrows(account)
        );
    }
```


