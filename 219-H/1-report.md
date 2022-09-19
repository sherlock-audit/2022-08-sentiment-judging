__141345__
# ERC20 tokens with pause mode could break repay/liquidation functionality

## Summary

Some ERC20 have pause mode, if toggled on, all the `transfer()` and `transferFrom()` will revert, causing the other protocols fail to function if depending on ERC20 token transfer. The repay and liquidate functionality are within the affected scope, since many ERC20 (USDC/USDT/WBTC/MATIC/SNX/LIDO and many others) can be paused.

The protocol could become insolvent due to DoS break the liquidation mechanism. 

## Vulnerability Detail

Take USDT as an example:
```solidity
// USDT pause mode
// https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code
    function transfer(address _to, uint _value) public whenNotPaused {}

    function transferFrom(address _from, address _to, uint _value) public whenNotPaused {}
```

If the ERC20 token `transfer()` and `transferFrom()` functions were paused, or the contract being compromised, these transfer functions will revert, the entire `repay()` or `sweepTo()` function will also revert, causing DoS in liquidation/repay.

There are many tokens has this mode, such as USDC, WBTC, MATIC, SNX, LIDO, etc. Some tokens implement the `pause` mode with different name, such as "status", "transferEnabled", etc. Many of them are the main stream assets. 

Although for each asset, the chance is low that the pause mode is toggled on. But for an account with multiple assets/borrow arrays, the chances will be much larger. As long as any single one of the assets/borrow arrays got the issue, the whole liquidation/repay functions will be down.


## Impact

- liquidation and repay functionality will be break, due to DoS
- temporary freezing of funds
- protocol insolvency
- smart contract unable to operate for involved deposits and borrow

Assets can be used as collateral or borrowed assets, the impacts might be the following:

- borrowed assets
    - denial of service on repay and liquidation
    - if the collateral's market value tumbles, the pool would suffer a loss
- collateral assets
    - liquidation mechanism might be completely down

When used as borrowed assets, the temporary break could be mitigated by offering alternative approach for repayment and liquidation. But in the case of collateral assets, the consequences could be worse, because the contingency situation might also lead to collapse of market value. If at the same time the collateral is not redeemable, the liquidators are not likely motivated to help keep the protocol solvent.

Even though this kind of scenario is rare, it is not groundless, considering the regulatory risks of USDC and other DeFi protocols, unforeseen attack causing contract being compromised, etc. When those special modes are toggled on, the whole market and protocols could be on the edge of material loss. And since in sentiment many assets are bundled together, as long as any single one of those ever come into trouble, the protocol could have big loss.


#### Reference

USDC implementation:
https://etherscan.io/address/0xa2327a938febf5fec13bacfb16ae10ecbc4cbdcf#code

USDT implementation:
https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code

WBTC
https://etherscan.io/token/0x2260fac5e5542a773aa44fbcfedf7c193bc2c599#code

MATIC
https://etherscan.io/token/0x7d1afa7b718fb893db30a3abc0cfc608aacfebb0#code

SNX
https://etherscan.io/token/0xc011a73ee8576fb46f5e1c5751ca3b9fe0af2a6f#code

Lido
Lido DAO Token
transfersEnabled
https://etherscan.io/token/0x5a98fcbea516cf06857215779fd812ca3bef1b32#code


## Code Snippet

Repay and liquidation receive collateral both rely on the ERC20 `transfer()/transferFrom()`.
```solidity
// repay eventually call the following
// protocol\src\utils\Helpers.sol
    function withdraw(address account, address to, address token, uint amt) internal {
        if (!isContract(token)) revert Errors.TokenNotContract();
        (bool success, bytes memory data) = IAccount(account).exec(token, 0,
                abi.encodeWithSelector(IERC20.transfer.selector, to, amt));
        require(success && (data.length == 0 || abi.decode(data, (bool))), "TRANSFER_FAILED");
    }

// protocol\src\core\AccountManager.sol
    function _liquidate(address _account) internal {
        // ...

        for(uint i; i < borrowLen; ++i) {
            address token = accountBorrows[i];
            LToken = ILToken(registry.LTokenFor(token));
            LToken.updateState();
            amt = LToken.getBorrowBalance(_account);
            token.safeTransferFrom(msg.sender, address(LToken), amt);
            LToken.collectFrom(_account, amt);
            account.removeBorrow(token);
        }
        account.sweepTo(msg.sender);
    }


// protocol\src\core\Account.sol
    function sweepTo(address toAddress) external accountManagerOnly {
        uint assetsLen = assets.length;
        for(uint i; i < assetsLen; ++i) {
            assets[i].safeTransfer(
                toAddress,
                assets[i].balanceOf(address(this))
            );
            hasAsset[assets[i]] = false;
        }
        delete assets;
        toAddress.safeTransferEth(address(this).balance);
    }
```



## Tool used

Manual Review

## Recommendation

- provide alternative options to repay
Such as repay/receive collateral the other assets instead of the specific ones.

- provide methods to repay for others

- provide alternative options to liquidate.
Temporarily just update the inner accounting of borrow amount, collateral amount, transfer the debt and collateral to the liquidator, and defer the real transfer of underlying assets. The liquidator can redeem later when everything resume normal operation. For instance, the borrower has 2400 MATIC as collateral and 2,000 USDC loan in debt, liquidator has 10,000 USDC on balance. If the collateral and debt are moved to the liquidator's account, although the transfer of USDC is paused, the overall health score of the liquidator account is still good, and the debt of the original borrower is cleared.

- insurance
The following insurance protocols have products covering some of the tokens referred: unslashed, nexusmutual, insurace, bridgemutual. As the TVL in Fringe is expected to grow even larger, buying multiple insurance policies for over protection might be an approach.
