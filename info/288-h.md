HonorLt
# A change in the registry might corrupt the system

## Summary
Updating addresses in the registry can have serious negative consequences.

## Vulnerability Detail
Registry allows the admin to update addresses, e.g. ```setLToken```, ```setAddress```. Then these changes can be synced in other contracts via ```initDep```. 

While I believe the intention was to give the system some flexibility, in reality updating the addresses in the registry might corrupt the system and its data. For example, the user has borrowed a token but can't repay it anymore because this token was deactivated:
```solidity
    function repay(address account, address token, uint amt)
        public
        onlyOwner(account)
    {
        ILToken LToken = ILToken(registry.LTokenFor(token));
        ...
        if (address(LToken) == address(0))
            revert Errors.LTokenUnavailable();
```
Or liquidation might fail:
```solidity
  for(uint i; i < borrowLen; ++i) {
    address token = accountBorrows[i];
    LToken = ILToken(registry.LTokenFor(token));
    ...
```
Or ```RiskEngine``` might miscalculate the user's total balance/borrows if the LToken address is updated.
Thus, I believe old borrowers should stick with the old configuration if possible.


Another problem is with the account manager. ```Account``` has ```accountManager``` that cannot be updated. When users want to repay the debt, they call ```function repay``` in the ```AccountManager``` contract. Repay calls ```LToken.collectFrom(account, amt)```. The ```collectFrom``` can only be called by account manager:
```solidity
    function collectFrom(address account, uint amt)
        external
        accountManagerOnly
```
If the manager is updated in the registry and synced:
```solidity
    function initDep(string calldata _rateModel) external adminOnly {
        rateModel = IRateModel(registry.getAddress(_rateModel));
        accountManager = registry.getAddress('ACCOUNT_MANAGER');
    }
```
It will become impossible to repay the debt of accounts that are managed by the old manager.

## Code Snippet
```solidity
    /**
        @notice Sets contract address for a given contract id
        @dev If address is 0x0 it removes the address from keys.
        If addressFor[id] returns 0x0 then the contract id is added to keys
        @param id Contract name, format (REGISTRY, RATE_MODEL)
        @param _address Address of the contract
    */
    function setAddress(string calldata id, address _address)
        external
        adminOnly
    {
        if (addressFor[id] == address(0)) {
            if (_address == address(0)) revert Errors.ZeroAddress();
            keys.push(id);
        }
        else if (_address == address(0)) removeKey(id);

        addressFor[id] = _address;
    }

    /**
        @notice Sets LToken address for a specified token
        @dev If underlying token is 0x0 LToken is removed from lTokens
        if the mapping doesn't exist LToken is pushed to lTokens
        if the mapping exist LToken is updated in lTokens
        @param underlying Address of token
        @param lToken Address of LToken
    */
    function setLToken(address underlying, address lToken) external adminOnly {
        if (LTokenFor[underlying] == address(0)) {
            if (lToken == address(0)) revert Errors.ZeroAddress();
            lTokens.push(lToken);
        }
        else if (lToken == address(0)) removeLToken(LTokenFor[underlying]);
        else updateLToken(LTokenFor[underlying], lToken);

        LTokenFor[underlying] = lToken;
    }
```

## Tool used

Manual Review

## Recommendation
The registry should be updated with special care if possible leaving the possibility for the users to stick with the old config or migrate the assets.
