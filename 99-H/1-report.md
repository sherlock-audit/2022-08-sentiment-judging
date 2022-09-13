PwnPatrol
# Liquidation can be blocked by getting blacklisted in USDC or USDT

## Summary
Attacker can get full protection against account liquidation.

## Vulnerability Detail
Some popular ERC20 tokens implement a blacklisting feature. The most common examples are USDC and USDT.
Surprisingly, due to recent events with Circle & Tornado Cash, it's known that getting blacklisted is actually quite easy to do. And the future doesn't look promising on that front - it will likely get worse.

POC:
1. Attacker borrows 1 USDC
2. Attacker gets blacklisted by Circle. Sending back and forth 10 ETH is well more than enough to get blacklisted (and it doesn't cost attacker anything other than gas).
3. Attacker deposits 1M DAI
4. Attacker borrows WBTC (5x leverage)
5. if it goes up => profit
    if it goes down => you cant be liquidated

Points 3-5 are just an example. One can pull off any trading strategy with this hack. For example, go short or super long or play delta neutral high leverage correlated assets. Doesn't matter. The hack allows high-risk plays without fear of liquidation.

Whatsmore, hackers can forcefully blacklist someone by donating minimal ETH from Tornado Cash. 

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
Use try catch logic to filter out blacklisted error cases.
Implement partial liquidations (not sure if this is viable though - happy to work on fixes in the future)