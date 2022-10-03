HonorLt
# Cannot repay the native asset

## Summary
Functions ```settle``` and ```repay``` handle the native asset differently.

## Vulnerability Detail
Function settle assumes that the asset with an address(0) of a borrowed asset means the native asset (ETH). However, this will not work, because the repay function will revert when trying to cast token address(0) to ERC20:
```solidity
  account.withdraw(address(LToken), token, amt);
  ...
  if (IERC20(token).balanceOf(account) == 0)
```
Thus making the owner unable to repay the native asset, or a malicious user can borrow the native asset to make the account non-liquidatable.

## Code Snippet
```solidity
  function settle(address account) external onlyOwner(account) {
      address[] memory borrows = IAccount(account).getBorrows();
      for (uint i; i < borrows.length; i++) {
          uint balance;
          if (borrows[i] == address(0)) balance = account.balance; // here it assumes the native asset is address(0)
          else balance = borrows[i].balanceOf(account);
          if ( balance > 0 ) repay(account, borrows[i], type(uint).max); // however, repay does not respect that
      }
  }
```
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
        account.withdraw(address(LToken), token, amt); // will revert when called with address(0)
        if (LToken.collectFrom(account, amt))
            IAccount(account).removeBorrow(token);
        if (IERC20(token).balanceOf(account) == 0) // will revert when called with address(0)
            IAccount(account).removeAsset(token);
        emit Repay(account, msg.sender, token, amt);
    }
```

## Tool used

Manual Review

## Recommendation
Make sure that the whole system is coherent and handles the native asset respectively, e.g. if address(0) is always a special case for the native asset then these functions should behave accordingly.
