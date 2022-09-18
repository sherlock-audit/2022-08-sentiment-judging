rbserver
# It is possible that receiving contract is not able to receive ETH through its `receive` or `fallback` function when calling `Helpers.withdrawEth` or `Helpers.safeTransferEth` function

## Summary
When the user is represented by a contract, calling the `Helpers.withdrawEth` or `Helpers.safeTransferEth` function can revert if the receiving contract cannot receive ETH through its `receive` or `fallback` function. This causes the user to not be able to receive the ETH amount that she or he deserves.

## Vulnerability Detail
As shown in the Code Snippet section, the `Helpers.withdrawEth` function is executed when the `AccountManager.withdrawEth` function is called, and `Helpers.safeTransferEth` function is executed when the `sweepTo` or `redeemEth` function is called. When calling `Helpers.withdrawEth` or `Helpers.safeTransferEth`, if the receiving address is a contract that does not, intentionally or unintentionally, implement the `receive` or `fallback` function in a way that supports receiving ETH, or if calling the receiving contract's `receive` or `fallback` function executes complicated logics that cost much gas, then it is possible that these function calls could revert.

## Impact
Since calling `Helpers.withdrawEth` or `Helpers.safeTransferEth` reverts, calling `AccountManager.withdrawEth`, `sweepTo`, or `redeemEth` everytime would revert. As a result, the receiving contract is not able to receive the corresponding ETH amount and lose the amount that it actually deserves.

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/utils/Helpers.sol#L44-L47
```solidity
    function withdrawEth(address account, address to, uint amt) internal {
        (bool success, ) = IAccount(account).exec(to, amt, new bytes(0));
        if(!success) revert Errors.EthTransferFailure();
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/utils/Helpers.sol#L35-L38
```solidity
    function safeTransferEth(address to, uint256 amt) internal {
        (bool success, ) = to.call{value: amt}(new bytes(0));
        if(!success) revert Errors.EthTransferFailure();
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L145-L152
```solidity
    function withdrawEth(address account, uint amt)
        external
        onlyOwner(account)
    {
        if(!riskEngine.isWithdrawAllowed(account, address(0), amt))
            revert Errors.RiskThresholdBreached();
        account.withdrawEth(msg.sender, amt);
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L163-L174
```solidity
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

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LEther.sol#L47-L53
```solidity
    function redeemEth(uint shares) external {
        uint assets = previewRedeem(shares);
        _burn(msg.sender, shares);
        emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
        IWETH(address(asset)).withdraw(assets);
        msg.sender.safeTransferEth(assets);
    }
```

## Tool used
Manual Review

## Recommendation
When the receiving contract is unable to receive ETH through its `receive` or `fallback` function as described above, WETH can be used to deposit the corresponding ETH amount, and the deposited amount can be transferred to the receiving contract.