0x52
# BalancerController.sol allows any LP to be burned which can be used to add malicious assets

## Summary

BalancerController.sol optimistically trusts all LP tokens which can be used to add malicious tokens to account.assets which can be used to avoid liquidation.

## Vulnerability Detail

[BalancerController.sol#L137-L141](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/balancer/BalancerController.sol#L137-L141)

        return (
            true,
            tokensIn,
            tokensOut
        );

[AccountManager.sol#L300-L305](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L300-L305)

    address[] memory tokensIn;
    address[] memory tokensOut;
    (isAllowed, tokensIn, tokensOut) =
        controller.canCall(target, (amt > 0), data);
    if (!isAllowed) revert Errors.FunctionCallRestricted();
    _updateTokensIn(account, tokensIn);

[AccountManager.sol#L347-L355](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L347-L355)

    function _updateTokensIn(address account, address[] memory tokensIn)
        internal
    {
        uint tokensInLen = tokensIn.length;
        for(uint i; i < tokensInLen; ++i) {
            if (IAccount(account).hasAsset(tokensIn[i]) == false)
                IAccount(account).addAsset(tokensIn[i]);
        }
    }

[Account.sol#L100-L103](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L100-L103)

    function addAsset(address token) external accountManagerOnly {
        assets.push(token);
        hasAsset[token] = true;
    }

BalancerController.sol#canExit always returns true for any balancer LP token, meaning that the user can burn any LP token. All tokens returned in tokensIn are added to account.asset. This allows the user to add any token they wish to account.assets because they can pair any arbitrary tokens they want in an LP pair send it to their account then use the account to burn them. Using this method a user could add a malicious token that will revert when the user is being liquidated. 

[AccountManager.sol#L367-L385](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L367-L385)

    function _liquidate(address _account) internal {
        IAccount account = IAccount(_account);
        address[] memory accountBorrows = account.getBorrows();
        uint borrowLen = accountBorrows.length;


        ILToken LToken;
        uint amt;


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

[Account.sol#L163-L174](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L163-L174)

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

When _liquidate is called, ALL tokens in account.assets are swept to msg.sender by account.sol#sweepTo. When it gets to the malicious asset it will always revert, making it impossible for the account to be liquidated.

## Impact

Liquidation of an account can be completely avoided

## Code Snippet

https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/balancer/BalancerController.sol#L107-L142

## Tool used

Manual Review

## Recommendation

BalancerController.sol#canExit should check that the LP being burned is an LP token supported by the system