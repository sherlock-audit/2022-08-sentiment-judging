PwnPatrol
# Users can avert liquidation by getting blacklisted in USDC or USDT

## Summary
Attacker can get full protection against account liquidation by holding a small amount of USDC and getting intentionally blacklisted.

## Vulnerability Detail
Some popular ERC20 tokens implement a blacklisting feature. The most common examples are USDC and USDT.
Due to recent events with Circle & Tornado Cash, it appears that getting blacklisted is quite easy to do. And the future doesn't look promising on that front - it will likely get worse.

POC:
1. Attacker borrows 1 USDC
2. Attacker gets blacklisted by Circle. Sending back and forth 10 ETH is well more than enough to get blacklisted (and it doesn't cost attacker anything other than gas).
3. Attacker deposits 1M DAI
4. Attacker borrows WBTC (5x leverage)
5. If it goes up => profit
    If it goes down => you cant be liquidated

Points 3-5 are just an example. One can pull off any trading strategy with this hack. For example, go short or super long or play delta neutral high leverage correlated assets. Regardless of the technique, the hack allows high-risk plays without fear of liquidation.

## Impact
Users may be able to forcefully create situations where they can't be liquidated, essentially gaming the protocol for asymmetric bets (while harming lenders whose funds remain tied up without interest payments ever being made).

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-PwnPatrol0x/blob/e6ef85090b220beef6ee51ac1dbecbf98dce6305/protocol/src/core/AccountManager.sol#L367-L384

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

https://github.com/sherlock-audit/2022-08-sentiment-PwnPatrol0x/blob/e6ef85090b220beef6ee51ac1dbecbf98dce6305/protocol/src/core/Account.sol#L163-L174

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

## Tool used

Manual Review

## Recommendation
Either implement try-catch logic to filter out blacklisted error cases in the _liquidate function, or implement partial liquidation functionality.