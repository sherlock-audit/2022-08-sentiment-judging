rbserver
# Lenders could lose native ETH amounts that they lent out because borrowers cannot successfully repay these given that there is an `LToken` contract for native ETH

## Summary
Lenders could lose the native ETH amounts that they lent out because these amounts cannot be repaid successfully by the borrowers given that there is an `LToken` contract for the native ETH.

## Vulnerability Detail
As the Code Snippet section shows, calling the `settle` function is called further calls the `repay` function. When `repay` is called, `account.withdraw(address(LToken), token, amt)` is executed, which further executes `(bool success, bytes memory data) = IAccount(account).exec(token, 0, abi.encodeWithSelector(IERC20.transfer.selector, to, amt))`. Then, when the `Account.exec` function is called, `(bool success, bytes memory retData) = target.call{value: amt}(data)` is executed.

When one of the borrowed tokens is the native ETH, calling `settle` would call `repay` with `borrows[i]` being `address(0)` as `repay`'s `token` input. Similarly, `repay` can be called directly with its `token` input being `address(0)` for repaying the borrowed native ETH amount. However, when the `token` input is `address(0)`, calling `account.withdraw` and then `IAccount(account).exec` would eventually execute `target.call{value: amt}(data)`, where `target` is `address(0)` and `data` is for calling an ERC20's `transfer` function. In this case, although executing `target.call` would not revert, none of the borrowed native ETH amount is returned to the `LToken` contract that corresponds to the native ETH given that this `LToken` contract exists.

## Impact
As mentioned above, given that the `LToken` contract exists for lending the native ETH to an account, calling `settle` or `repay` for repaying the borrowed native ETH amount for the account does not return the native ETH amount to the `LToken` contract. As a result, the lenders lose these native ETH amounts that they lent out because calling `settle` or `repay` cannot successfully repay these amounts by the borrowers.

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L318-L326
```solidity
    function settle(address account) external onlyOwner(account) {
        address[] memory borrows = IAccount(account).getBorrows();
        for (uint i; i < borrows.length; i++) {
            uint balance;
            if (borrows[i] == address(0)) balance = account.balance;
            else balance = borrows[i].balanceOf(account);
            if ( balance > 0 ) repay(account, borrows[i], type(uint).max);
        }
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L227-L242
```solidity
    function repay(address account, address token, uint amt)
        public
        onlyOwner(account)
    {
        ILToken LToken = ILToken(registry.LTokenFor(token));
        if (address(LToken) == address(0))
            revert Errors.LTokenUnavailable();
        LToken.updateState();
        if (amt == type(uint256).max) amt = LToken.getBorrowBalance(account);
        account.withdraw(address(LToken), token, amt);
        if (LToken.collectFrom(account, amt))
            IAccount(account).removeBorrow(token);
        if (IERC20(token).balanceOf(account) == 0)
            IAccount(account).removeAsset(token);
        emit Repay(account, msg.sender, token, amt);
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/utils/Helpers.sol#L49-L54
```solidity
    function withdraw(address account, address to, address token, uint amt) internal {
        if (!isContract(token)) revert Errors.TokenNotContract();
        (bool success, bytes memory data) = IAccount(account).exec(token, 0,
                abi.encodeWithSelector(IERC20.transfer.selector, to, amt));
        require(success && (data.length == 0 || abi.decode(data, (bool))), "TRANSFER_FAILED");
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L149-L156
```solidity
    function exec(address target, uint amt, bytes calldata data)
        external
        accountManagerOnly
        returns (bool, bytes memory)
    {
        (bool success, bytes memory retData) = target.call{value: amt}(data);
        return (success, retData);
    }
```

## Tool used
Manual Review

## Recommendation
Separate logics for paying back the borrowed native ETH amount can be added in the `repay` function. When `token` is `address(0)`, these logics can be executed, instead of executing `account.withdraw` unconditionally, when calling `repay`.