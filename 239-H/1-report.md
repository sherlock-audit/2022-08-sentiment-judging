Bahurum
# ERC777 reentrancy in self-liquidation

## Summary
Attacker can liquidate his own position. If there is a borrowed token or collateral that is ERC777, this allows him to reenter and re-borrow before `sweepTo` sends him all the collateral + borrowed, despite the debt > 0.

## Vulnerability Detail
There are two lines of code from wich an ERC777 allows reentrancy:
- [AccountManager.sol#L380](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L380): if a borrowed asset is ERC777 token.
- [Account.sol#L166](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L166): if a collateral asset is ERC777

0. Attacker deposits a tiny amount of an allowed ERC777 token with symbol XYZ
1. Attacker deposits 1000 DAI 
2. Attacker borrows 4999 DAI
3. Attacker waits for interest to increase his debt to 5000 DAI
4. Attacker liquidates himself  
    During liquidation:  
   4.1 Attacker pays back debt of 5000 DAI  
   4.2 In `sweepTo`: XYZ token collateral are sent back to attacker
   4.3 Attacker reacts to the transfer by calling `borrow()` and borrowing 4999 DAI
   4.4 `sweepTo` continues execution: 1000 + 4999 DAI are sent to attacker
5. Attacker has 5999 DAI after the attack (stole 4999 DAI)

This can be repeated many times to multipy each time the amount stolen by 5, and with different tokens too, so it allows to steal a very large amount of liquidity from the protocol.

## Impact
Entire protocl can be drained.

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L367-L385

```solidity
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
```

## Tool used

Manual Review

## Recommendation

Check that debt is 0 and close account before call to `sweepTo()` to deny re-borrowing.

```diff
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
+       if (!account.hasNoDebt()) revert Errors.OutstandingDebt();
+       account.deactivate();
+       registry.closeAccount(_account);
+       inactiveAccountsOf[msg.sender].push(_account);
        account.sweepTo(msg.sender);
+       emit AccountClosed(_account, msg.sender);
    }
```