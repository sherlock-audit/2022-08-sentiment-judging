HonorLt
# Uncapped arrays

## Summary
```assets``` and ```borrows``` do not have upper boundaries leaving a gap for a potential denial of service attack. 

## Vulnerability Detail
```Account``` has two arrays, ```assets``` and ```borrows```. Those arrays have no max limits, thus making it possible they grow so large that the operations iterating over them will fail due to reaching the block gas limit. There are many loops that rely on the length of these arrays, for instance, when liquidating it first iterates over ```accountBorrows.length```, and then ```assets.length``` when sweeping the assets. If there are no restrictions on how many different tokens an account can have/borrow, and there will be many tokens whitelisted, a malicious actor can borrow each asset a little bit making the account non-liquidatable.
The possibility of such an attack depends on how you plan to manage the protocol, and how many tokens are supposed to be supported but I believe it is always a good practice to take into consideration and enforce such limits in the contracts.

## Code Snippet
```solidity
    function addAsset(address token) external accountManagerOnly {
        assets.push(token);
        hasAsset[token] = true;
    }

    function addBorrow(address token) external accountManagerOnly {
        borrows.push(token);
    }
```
```solidity
    function settle(address account) external onlyOwner(account) {
        address[] memory borrows = IAccount(account).getBorrows();
        for (uint i; i < borrows.length; i++) {
           ...
        }
    }

  function _getBorrows(address account) internal view returns (uint) {
         ...
        address[] memory borrows = IAccount(account).getBorrows();
        uint borrowsLen = borrows.length;
        ...
        for(uint i; i < borrowsLen; ++i)
        ...
 }

  function _getBalance(address account) internal view returns (uint) {
        address[] memory assets = IAccount(account).getAssets();
        uint assetsLen = assets.length;
        uint totalBalance;
        for(uint i; i < assetsLen; ++i) {
        ...
  }

  function _liquidate(address _account) internal {
        ...
        address[] memory accountBorrows = account.getBorrows();
        uint borrowLen = accountBorrows.length;
        ...
        for(uint i; i < borrowLen; ++i) {
        ...
 }
```

## Tool used

Manual Review

## Recommendation
Consider adding an upper limit for how many ```assets``` / ```borrows``` an account can have at maximum, so in case this limit is reached, functions depending on it will not exceed the block gas limit.
