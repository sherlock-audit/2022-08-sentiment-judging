Lambda
# AccountManager: approve can be abused to drain an account by using UniswapV3Router

## Summary
An account can `approve` UniswapV3Router, which then can drain the account.

## Vulnerability Detail
It is only possible to `approve` addresses where a corresponding controller exists. However, one such address will be the `UniswapV3Router` (because it is intended to be used in other places) which provides the `multiCall` function. This can be abused like this:
1.) Account A approves `UniswapV3Router` (via `AccountManager.approve`) for some ERC4626 vault. This succeeds because there is a controller for `UniswapV3Router`.
2.) Account A mints those ERC4626 tokens and takes out a loan. Because of the ERC4626 tokens, his health ratio is 1.2
3.) Some EOA B uses `UniswapV3Router.multicall` to call `withdraw(balanceOf(A), address(B), address(a))` for him. Because `UniswapV3Router` is approved to spend those tokens, this succeeds. After this call, the account of A is now drained and has a health ratio of 0. All assets belong to the EOA B (which will in practice be owned by the same person as the account A).

## Impact
See vulnerability details, an account can be completely drained like this.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-OpenCoreCH/blob/015efc78e890daa1cf640d92125608f22cf167ed/protocol/src/core/AccountManager.sol#L277

https://github.com/Uniswap/v3-periphery/blob/8264326f537df1695b2074ccd5a772b48b5d7e5e/contracts/lens/UniswapInterfaceMulticall.sol#L27

## Tool used

Manual Review

## Recommendation
Do not allow approving `UniswapV3Router`.