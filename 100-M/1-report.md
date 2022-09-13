PwnPatrol
# Liquidation and Settlement Flows can be blocked by Pausable ERC20 tokens

## Summary
Pasued tokens pose an insolvency risk to the system and can break the Liquidation Flow

## Vulnerability Detail
Some popular ERC20 tokens implement a pausable feature. The most common examples are BNB.
It is a rare event but when it happens it breaks Liquidation and Settlement Flows. For an extended pause period, it can render Liquidation Flow unprofitable for Liquidators.

The probability of such an event is low and cannot be triggered by Attacker. On the other hand, the severity of such an event is high thus I'll mark this as Medium. 

## Impact
Protocol Insolvency Risk.
Liquidation Flow DoS.

## Code Snippet
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
            token.safeTransferFrom(msg.sender, address(LToken), amt); // reverts here
            LToken.collectFrom(_account, amt);
            account.removeBorrow(token);
        }
        account.sweepTo(msg.sender);
    }
```

## Tool used

Manual Review

## Recommendation
Use try-catch logic to filter out pausable error cases.
Implement partial liquidations (not sure if this is viable though - happy to work on fixes in the future)