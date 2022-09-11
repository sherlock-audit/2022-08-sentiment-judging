grhkm
# liquidation logic will not seize user balance

## Summary
If an account becomes unhealthy, liquidation logic will not seize user balance, instead requires liquidation function caller to repay the amount and then send all the amount back to account owner which is incorrect

## Vulnerability Detail
1. Account A becomes unhealthy due to sudden price drop in one of its token

2. Account A total balance of all token say becomes 50 and total borrows becomes 80

3. Obviously Account A is not interested in liquidation as then he would need to pay more (liquidation requires to pay all debt before moving all assets back to account owner)

4. Now this account is stuck since using liquidate function does not make sense for contract owner also since then contract owner will need to repay loan and account balance will be transferred to Account A which does not make sense

```
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

## Impact
There is no proper way for Admin to seize user account assets in case of liquidation

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L250

## Tool used
Manual Review

## Recommendation
Add user blacklisting for users who are always falling unhealthy. Also add a method which allows contact owner to seize user account balance to himself in case account remains unhealthy for x days