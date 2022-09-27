kirk-baird
# MED/HIGH: Reentrancy In UniSwap or SushiSwap Can Liquidate Users

## Summary

If an attacker were to gain control of execution during any uniswap or sushiswap swap call the calling account may not be in a healthy position as the swap is partially complete with the account owning neither `tokensIn` nor `tokensOut` but instead some intermediary token. Since they are unhealthy and an attacker has control of execution this user may be liquidated.

A user may also perform this attack on themselves to liquidate themselves and reenter other functions such as `borrow()`, `deposit()` or `withdraw()`.

## Vulnerability Detail

The swap functions in Uniswap and SushiSwap may take a `path` parameter. This allows swapping through a chain of multiple different tokens. e.g. USDC -> WETH -> DAI. An attacker may create a pair which provides a beneficial swap though a malicious token that gives them control of execution. e.g. USDC -> MALICIOUS -> DAI. 

In this instance the swap will have `Account.exec()` will have `tokensIn[] = [DAI]` and `tokensOut = [USDC]`. Since the attacker has control during `MALICIOUS` step and the user has already transferred the `USDC` to uniswap/sushiswap but not yet received `DAI` `RiskEngine.isAccountHealthy()` may return false (depending on their other positions but is more likely since they're missing some of their collateral).

The attacker with control could call `AccountManager.liquidate()` on the `Account` claiming all the remaining ERC20 tokens at the cost of the borrows.

The attacker would then let the swap complete and `Account.exec()` will complete since there are no longer any borrows the `Account` will be healthy. 

## Impact

It's possible to reenter a range of different functions. The attack described above could be used to liquidate users during a swap under certain conditions. 

## Code Snippet

```solidity
    function exec(
        address account,
        address target,
        uint amt,
        bytes calldata data
    )
        external
        onlyOwner(account)
    {
        bool isAllowed;
        address[] memory tokensIn;
        address[] memory tokensOut;
        (isAllowed, tokensIn, tokensOut) =
            controller.canCall(target, (amt > 0), data);
        if (!isAllowed) revert Errors.FunctionCallRestricted();
        _updateTokensIn(account, tokensIn);
        (bool success,) = IAccount(account).exec(target, amt, data);
        if (!success)
            revert Errors.AccountInteractionFailure(account, target, amt, data);
        _updateTokensOut(account, tokensOut);
        if (!riskEngine.isAccountHealthy(account))
            revert Errors.RiskThresholdBreached();
    }
```

## Tool used

Manual Review

## Recommendation

Consider adding a reentrancy guard such the one provided by [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol) on each of the following functions:
- `openAccount()`
- `closeAccount()`
- `depositEth()`
- `withdrawEth()`
- `deposit()`
- `withdraw()`
- `borrow()`
- `repay()` (only if no called by `settle()`, may need an internal helper here)
- `liquidate()`
- `approve()`
- `exec()`
- `settle()`
