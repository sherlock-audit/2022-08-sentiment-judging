rvierdiiev
# Registry.init and AccountManager.init could be frontrunned

## Summary
`Registry` contract has `init` [function](https://github.com/sherlock-audit/2022-08-sentiment-rvierdiyev/blob/main/protocol/src/core/Registry.sol#L61) that doesn't have any protection from being run by anyone. Same is for `AccountManager.init` [function](https://github.com/sherlock-audit/2022-08-sentiment-rvierdiyev/blob/main/protocol/src/core/AccountManager.sol#L68-L73).

Attacker can call these functions before the sponsors and then he will be owner of proxy contracts. New deployment will be needed.
In case if `AccountManager.init` will be front runned and sponsors won't notice that, then attacker can call `AccountManager.initDep` [function](https://github.com/sherlock-audit/2022-08-sentiment-rvierdiyev/blob/main/protocol/src/core/AccountManager.sol#L76-L81) to initialize dependencies. And if such malicious contract will be used further then attacker as an owner will be able to call `toggleCollateralStatus` [function](https://github.com/sherlock-audit/2022-08-sentiment-rvierdiyev/blob/main/protocol/src/core/AccountManager.sol#L395-L397) and add any collateral he wants.

## Tool used

Manual Review

## Recommendation
Make sure that no one can front run you. For example it's possible to create immutable variable `initializer` and set it in constructor to caller. Then make function `init` to be called only by `initializer`.