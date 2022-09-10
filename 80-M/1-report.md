cergyk
# Native funds mistakenly sent to LEther will be permanently frozen

## Summary

LEther accepts native funds sent over without an existing function call.

All native funds sent to the contract this way will be frozen permanently.

## Vulnerability Detail

The only native tokens utilizing business logic in LEther is depositEth(), that operates `msg.value` only. Parent LToken and ERC4626 contracts do not deal with native funds.

There is no way to retrieve native funds directly sent to LEther contract and there is no business logic utilizing them, so `receive() external payable` provides a way for a permanent fund freeze and no utility.

## Impact

Setting the severity to **medium** as this is conditional on user/downstream system mistake of directly sending the native funds or calling a non-existing signature with the native funds attached.

Impact, however, is the loss of these funds for such users.

## Code Snippet

The only native funds operating business logic is depositEth() that deals with `msg.value` only and cannot reach any other native funds on the contract balance:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LEther.sol#L31-L38

Notice that total assets of the contract is determined with inherited LToken's totalAssets(), which uses WETH balance:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LToken.sol#L185-L193

Native funds sent directly aren't deposited to WETH and do not increase total assets this way, so the shareholders have no access to such funds either.

I.e. any funds that end up on LEther balance via general receive() will become permanently frozen.

## Recommendation

Consider removing the receive() as it's not used in the logic:

https://github.com/sherlock-audit/2022-08-sentiment-dmitriia/blob/9745a9a32641e4a709dcf771bfcfb97cb1e89bd9/protocol/src/tokens/LEther.sol#L55

```solidity
-   receive() external payable {}
```