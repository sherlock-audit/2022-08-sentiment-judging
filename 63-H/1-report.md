grhkm
# Re-entrancy with multiple smaller bugs can be used to borrow large amount and lock it in the account

## Summary

Using re-entrancy and multiple other smaller bugs it is possible to set `account.hasAsset[anyAsset] = true` while account has no assets at all. As borrow checks for `hasAsset` and only adds asset if `hasAsset = false`, borrow of such asset will succeed, the asset will be transferred to account, but won't be added to account's `assets` array. Since all account balance calculations only sum up assets in the `assets` array, the borrowed asset won't count and account will be immediately liquidatable. However, liquidator will NOT get that borrowed asset for the same reason (because it's not in the `assets` array), which will effectively lock the borrowed amount in the account with only the attacker being able to access it if needed (he can't withdraw it, but he can use it in the other ways to profit from this at the expense of `LToken` lenders, who won't be able to withdraw all deposited funds).

## Vulnerability Detail

There are multiple bugs in the code, each of them can not be used to cause harm to the protocol. However, when multiple bugs are combined, it becomes possible to exploit the protocol.

List of bugs used in the attack:

1. Uniswap v2 controller in `removeLiquidity` doesn't check if tokens received are allowed. It seems to assume that account can only have allowed uni v2 lp tokens (as it's checked in `addLiquidity`), however any lp tokens can easily be transferred to account directly (not via `exec`). This makes it possible to call `exec` to `removeLiquidity` and add ANY tokens to account's assets list (although `accountManager.exec` will fail because no oracle is set up for these tokens)

https://github.com/sherlock-audit/2022-08-sentiment-panprog/blob/6da8a0e43d272eda0d40760cd90c50ed7ce21fee/controller/src/uniswap/UniV2Controller.sol#L182-L189

2. Re-entrancy inside `accountManager.exec` is possible via custom tokens, as `account.exec` calls external functions, which can call functions (such as `transfer`) on custom tokens, deployed by malicious users. This can be combined with bug 1 to do malicious stuff and then remove custom token from assets in re-entrancy to allow transaction to finish successfully.

https://github.com/sherlock-audit/2022-08-sentiment-panprog/blob/6da8a0e43d272eda0d40760cd90c50ed7ce21fee/protocol/src/core/AccountManager.sol#L306

3. `Account.sweepTo` is very vulnerable to re-entrancy. While it assumes that asset tokens are ERC20, bugs 1 and 2 let the user add custom token to `assets`, do `sweepTo` (via liquidation or account closure) and re-enter via custom token's `transfer` function. Moreover, inside the `sweepTo`'s loop `hasAsset` is set to `false`, but asset is not removed from the list as the whole assets are deleted after the loop, desyncing `hasAsset` from the `assets` list.

https://github.com/sherlock-audit/2022-08-sentiment-panprog/blob/6da8a0e43d272eda0d40760cd90c50ed7ce21fee/protocol/src/core/Account.sol#L163-L174

Steps to achieve borrowed assets lock:

1. OpenAccount
2. Create your own CustomERC20 token `HackToken`
3. Deploy uniswap v2 pool `USDC-HackToken`, add liquidity with minimum `USDC`
4. Send uniswap v2 LP token to account's address
5. Call `AccountManager.exec` to `removeLiquidity` from uniswap v2 LP token, abusing bug 1 and receiving small `USDC+HackToken` (`tokensIn` are set and both `USDC` and `HackToken` are added to account's `assets`)
7. Still inside `exec` (in `Account.exec`), use bug 2 to re-enter via `HackToken.transfer` function (called by uniswap pair's `burn`)
7. in `HackToken.transfer` do re-entrancy by calling `AccountManager.closeAccount`, which performs `account.sweepTo` as the last step, where:
  7.1. `assetLen = 2` (`USDC`, `HackToken`)
  7.2. when `i = 0`, `USDC` is transferred
  7.3. when `i = 1`, `HackToken`'s `safeTransfer` is called, which uses bug 3 to do another reentrancy
    7.3.1. in `HackToken`'s `safeTransfer`, first call `AccountManager.openAccount` to re-gain access to it (since it's closed) - this restores account access
    7.3.2. call `AccountManager.deposit` `DAI` with any `DAI` amount (can be `0`). This sets `hasAsset[DAI]` to true and adds `DAI` to `assets` at `index = 2` (assets now has `USDT`, `HackToken`, `DAI`)
    7.3.3. exit reentrancy
  7.4. still for `i = 1`, `hasAsset[HackToken]` is set to `false`
  7.5. for loop finishes even though there are now 3 assets, because for loop length `assetLen` is stored in local variable before the `assets` change
  7.6. `delete assets;` is done to clear all 3 assets. However, `hasAsset[DAI]` is still set to true
  7.7. exit re-entrancy
8. `exec` finishes
9. transaction finishes.
10. At this point we have a clean account except `hasAsset[DAI]` is set to `true`, while there are no assets.
11. deposit some token (other than `DAI`), for example `1000 USDC`
12. borrow `4000 DAI` against `1000 USDC`. `isBorrowAllowed` returns true, because (`assets=1000+4000=5000`, `borrow=4000`, `5000/4000 = 1.25 > 1.2`). However, `hasAsset[DAI]` is `true` and thus `IAccount(account).addAsset(token);` is not called, so `DAI` is NOT added as an asset. `4000 DAI` is still transferred from `DAI LToken` to account though.

https://github.com/sherlock-audit/2022-08-sentiment-panprog/blob/6da8a0e43d272eda0d40760cd90c50ed7ce21fee/protocol/src/core/AccountManager.sol#L210-L213

13. At this point account has `1000 USDC + 4000 DAI` and `4000 DAI` debt, however account.assets array has only 1 item - `USDC` (no `DAI`!). Since account health is determined based on `assets` array only, `RiskEngine.getBalance` for account will return `1000` (`USDC` balance), and so health factor is `1000/4000 = 0.25`, well below the threshold.
14. At this point account is subject to liquidation, however any liquidator will only be able to get `1000 USDC` while having to pay `4000 DAI` debt. As such, liquidators will not liquidate such account (and if they do, we can then safely withdraw our `4000 DAI`). And until account is liquidated, `4000 DAI` of borrowed amount is locked.

## Impact

Borrowed amount at max leverage is locked from access of anybody but the account owner (attacker). Liquidators can not get this amount. This makes it possible to use `1/5` of `LToken` assets to lock all `LToken` assets permanently, meaning `LToken` lenders will not be able to withdraw any of their assets.

It is also possible for attacker to perform this attack for his economic benefit at the expense of `LToken` lenders. One of the possible scenarios is as following:

1. `ETH = $1000`
2. make `hasAsset[USDT] = true` with no assets as described in exploit above
3. Deposit `1000 USDC`
4. Borrow `2 WETH` against `1000 USDC`, then sell `2 WETH` for `2000 USDC` (assets=3000, debt=2000, health=1.5)
5. Borrow `2000 USDT`, which is still allowed (assets=3000+2000=5000, debt=2ETH+2000USDT=4000, health = 1.25), however right after borrow we have `2000 USDT` (hidden as it's not in assets), 3000 USDC in assets and 2 WETH + 2000 USDT = 4000 in debt (so health = 3000/4000 = 0.75, subject to liquidation, which is very not profitable).
6. Use the additional borrow as a protection against the liquidation while basically keeping leveraged short `2 ETH` position for free.
7. In another account set `hasAsset[USDT] = true` with no assets
8. Swap `500 USDC` for `0.5 WETH`, deposit into account 2.
9. Borrow `1000 USDC` against `0.5 WETH` and exchange into `1 ETH`
10. Borrow `1000 USDT`, which is allowed, but then hidden similar to account 1
11. At this point we have leveraged `2 WETH` short position and leveraged `1.5 WETH` long position with 1500 USDC of our own funds used.

Possible outcomes:
1. `ETH = $2000`: Long position has `+$2000` profit, short position can be abandoned for `-$1000` loss, a total of `+$1000` profit.
2. `ETH = $500`: Short position has `+$1000` profit, long position can be abandoned for `-$500` loss, a total of `+$500` profit.
3. If at any time position is liquidated, we are able to immediately withdraw the "hidden" amount, which will be our profit as it's larger than initial deposit. We will also be able to pay back the debt in another account and withdraw our assets if it still has more assets than debt. 

This is not the perfect scenario, but it demonstrates that the vulnerability can be used for economic advantage of the attacker at the expense of `LToken` lenders (because attacker will permanently lock one of the accounts with borrowed funds, so not all lenders will be able to withdraw their assets). There might be better and more profitable scenarios for the attacker.

## Code Snippet

Create folder `./protocol/src/test/integrations/attacks` and put these 3 test files there:

https://gist.github.com/panprog/d9994e624696dbb636102f0c58e7c470

## Tool used

Manual Review

## Recommendation

1. Add ReEntrancy guard from openzeppelin to main entry functions, mostly all functions in `AccountManager`.
2. Add token checks to uniswap v2 controller (and carefuly verify all the other controllers - for example, curve `StableSwap3PoolController` incorrectly uses pool address as token, but it should be `pool.token` instead).
3. I think that assets collateral handling in general is far from perfect - there are collateral tokens which are not respected by controllers at all (controllers can add their tokens to assets as they have separate allowed lists, but then these tokens will be used as collateral as collateral token is only checked at deposit, but not at exec). I suggest reviewing collateral handling and refactoring it. For example, when calculating account balance, use only collateral tokens for calculations, ignoring all the others.