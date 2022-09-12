icedpeachtea
# Account can be initialized multiple times if _accountManager is address(0)

## Summary
Account's contract `init` function can be reinit multiple times if `_accountManager` is `address(0)`.

## Vulnerability Detail
The if statement determines the `accountManager` is not `address(0)`, if yes it determines the contract is already initialized. However, it does not check whether the input `_accountManager` is `address(0)`. If it is, the `init` function can be called twice and other people (such as an attacker) can modify the `_accountManager` to his/her controlled address. 

https://twitter.com/Hacxyk/status/1529389391818510337

## Impact
A misconfiguration of setting `_accountManager` to `address(0)` would open up possibilities for attacker to reinitialize the contract. 

## Code Snippet
- protocol/src/core/Account.sol:59

```solidity
function init(address _accountManager) external {
        if (accountManager != address(0)) 
            revert Errors.ContractAlreadyInitialized();
        accountManager = _accountManager;
    }
```

## Tool used

Manual Review

## Recommendation
Prevent `_accountManager` to be `address(0)` when calling the function.

```solidity
function init(address _accountManager) external {
        require(_accountManager != address(0), "Cannot init as 0x0 address!");
        if (accountManager != address(0)) 
            revert Errors.ContractAlreadyInitialized();
        accountManager = _accountManager;
    }
```