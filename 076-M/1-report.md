panprog
# If oracle is set for ERC777 token, re-entrancy is possible to steal all LToken funds

## Summary

If oracle is set for any `ERC777` or similar token (tokens which call receiver's hook after receiving it), re-entrancy in `Account.sweepTo` allows to borrow funds, which are immediately withdrawn along with all account assets without any health checks, leaving account with 0 assets and big debt, making it possible to drain all LToken funds.

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/core/Account.sol#L163-L174

## Vulnerability Detail

If oracle is set for `ERC777` token, it's possible to add these tokens to assets (even if collateral or controller allowed token for them is not set, using the bug in uniswap v2 controller for `removeLiquidity`, which doesn't check if tokenIn is allowed).

Once `ERC777` token is added to assets, malicious user can close his account, which calls `Account.sweepTo`, which has multiple re-entrancy scenarios. As `ERC777` token will call user's hook after receiving these tokens, the following actions are possible in the hook to steal funds:

- User can borrow funds, exit reentrancy, and all borrowed tokens along with all account assets will be immediately transfered to user without any health checks.
- For tokens which were in `assets` list before the `ERC777` token, `hasAsset[]` is set to false even though the token is still in the assets list. Depositing these tokens again will add them to assets again, counting them twice in account balance calculations, which allows to borrow even more funds.
- Ether is transferred back to user after all the `ERC20` tokens, so ether can also be used to borrow funds against, and it will also be transerred back to user along with all the `ERC20` tokens.

List of bugs used in the attack:

1. Uniswap v2 controller in `removeLiquidity` doesn't check if tokens received are allowed. It seems to assume that account can only have allowed uni v2 lp tokens (as it's checked in `addLiquidity`), however any lp tokens can easily be transferred to account directly (not via `exec`). This makes it possible to call `exec` to `removeLiquidity` and add ANY tokens to account's assets list (and if oracle is set for the token, `accountManager.exec` will succeed as it doesn't check for asset tokens to be allowed as collateral)

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/controller/src/uniswap/UniV2Controller.sol#L182-L189

2. `Account.sweepTo` is very vulnerable to re-entrancy. While it assumes that asset tokens are ERC20, if ERC20-compatible tokens with callback hooks are allowed (such as `ERC777` tokens), these tokens can re-enter to steal funds.

https://github.com/sherlock-audit/2022-08-sentiment/blob/main/protocol/src/core/Account.sol#L163-L174

Steps to steal funds from LToken:

1. OpenAccount
2. If `ERC777` token is allowed as collateral, deposit `ERC777` token (such as `imBTC`, which is pegged 1:1 to BTC and is ERC777). If `ERC777` is not allowed as collateral, but the oracle for this token is set up, then use bug 1:
  2.1. Deploy uniswap v2 pool `USDC-imBTC`, add liquidity with minimum `USDC` and `imBTC`
  2.2. Send uniswap v2 LP token to account's address
  2.3. Call `AccountManager.exec` to `removeLiquidity` from uniswap v2 LP token, abusing bug 1 and receiving small amounts of `USDC+imBTC` (`tokensIn` are set and both `USDC` and `imBTC` are added to account's `assets`)
3. Deposit `0 DAI` to add `DAI` to account `assets` (this is to include `DAI` in `sweepTo` at the last index, which will make it possible to borrow `DAI` in re-entrancy and immediately receive all of it without health checks)
4. Close account
5. Close account will performs `account.sweepTo` as the last step, where `USDC` is transferred to user, then `imBTC` is transferred to user, which calls `tokensReceived` on user's contract allowing re-entrancy. In `tokensReceived`:
  5.1. Call `AccountManager.openAccount` to re-gain access to account (since it's closed) - this restores account access
  5.2. Deposit `1 ether` to account
  5.3. Borrow `4000 DAI`
  5.4. Exit re-entrancy
6. `sweepTo` continues and sends `4000 DAI` to user, then deletes `assets` and sends `1 ether` to user.
7. At this point user has received back its `1 ether` and `4000 DAI`, leaving account with `0` assets and `4000 DAI` debt.

This scenario can be made even more capital efficient to steal more money if user deposits `1000 USDC` instead of `1 ether`, which will add `USDC` to assets, making his balance `2000` instead of `1000`, allowing to borrow `8000 DAI` for the same deposited amount. If this is repeated a few times, it's possible to steal millions with only a small initial capital.

## Impact

If any `ERC777` (or similar) token oracle is added, it's very easy to steal all funds from all LTokens which you can borrow from.

## Code Snippet

Create folder `./protocol/src/test/integrations/attacks` and put these 2 test files there:

https://gist.github.com/panprog/2f16348325303869f7d84653cf99fba1

## Tool used

Manual Review

## Recommendation

1. Add ReEntrancy guard from openzeppelin to main entry functions, mostly all functions in `AccountManager`.
2. Add correct token checks to uniswap v2 controller.