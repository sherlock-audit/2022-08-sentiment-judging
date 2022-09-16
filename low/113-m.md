0xNazgul
# [NAZ-M6] User Could Change The State Of The System While Paused

## Summary
Several functions are calling `LToken.updateState()` function which will change the state of the system. 

## Vulnerability Detail
When the system is in Pause mode, the system state should be frozen. However, it was possible someone to call the `LToken.updateState()` function when paused, thus making changes to the system state.

## Impact
The point of a system being `pausable` is to provide your smart contract the ability of blocking calls of critical functions. The pattern helps prevent attackers from continuing their work and any further system state changes until the situation is fixed.

## Code Snippet
[`LToken.sol#L153`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/tokens/LToken.sol#L153), [`LToken.sol#L200`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/tokens/LToken.sol#L200), [`LToken.sol#L246`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/tokens/LToken.sol#L246), [`AccountManager.sol#L234`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/AccountManager.sol#L234), [`AccountManager.sol#L378`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/AccountManager.sol#L378)

## Tool used
Manual Review

## Recommendation
Add the `whenNotPaused()` modifier to the listed functions to prevent system state changes when `paused`.