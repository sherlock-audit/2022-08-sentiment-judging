minera
# Loss of Precision Bug

## Summary
- Manipulation of LToken/LEther when totalSupply is zero can lead to implicit minimum deposit amount and loss of user funds due to rounding errors

## Vulnerability Detail
- Where: [ERC4626](https://github.com/sherlock-audit/2022-08-sentiment-kankodu/blob/main/protocol/src/tokens/utils/ERC4626.sol)
- When: vault.totalSupply == 0
- Description:
    - When totalSupply is zero an attacker goes ahead and executes the following steps
        1. Deposit 1 wei of token to mint 1 wei of shares
        3. Transfer(Donate) z underlying tokens directly to vault address
                - This leads to 1 wei of vault share worth z+1 tokens
                - Attacker won't have any problem making this z as big as they want as they have all the claim to it as a holder of 1 Wei of vault share
    - This attack has two implications
        - Implicit minimum Amount and funds lost due to rounding errors
            - If an attacker is successful in making 1 wei of vault share worth z underlying tokens and a user tries to mint vault shares using k* z underlying tokens then,
                - If k<1, then the user gets zero vault shares and that fails with [ZERO_SHARES](https://github.com/sherlock-audit/2022-08-sentiment-kankodu/blob/main/protocol/src/tokens/utils/ERC4626.sol#L52) error.
                    - This leads to an implicit minimum amount for a user at the attacker's discretion.
                - If k>1, then users still get some vault shares but they lose (k- floor(k)) * z) of underlying tokens which get proportionally divided between vault share holders due to rounding errors.
                    - users keep loosing up to 25% of their underlying tokens. (see [here](https://www.desmos.com/calculator/tjz5j62fng) for visualisation)
                    - This means that for users to not lose value, they have to make sure that k is an integer.
## Impact
- If this attack is executed there is no other way to rectify it then deploying a new LToken altogether.
## Code Snippet
- Add below test in [LToken.t.sol](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/test/tokens/LToken.t.sol)
```
function testFailVictimInteraction() public {
        uint256 z = 1000 ether;
        erc20.mint(address(this), 2 * z + 1);

        //step 1: mint 1 wei of share when totalSupply is zero
        assert(lErc20.totalSupply() == 0);
        erc20.approve(address(lErc20), 1);
        lErc20.deposit(1, address(this));
        assert(lErc20.balanceOf(address(this)) == 1);

        //step2: donate z
        erc20.transfer(address(lErc20), z);

        //victim tries to mint with less than z+1 amount gets 0 shares which fails

        erc20.approve(address(lErc20), z);
        lErc20.deposit(z, address(this));
    }

```
## Tool used

Manual Review

## Recommendation
- I like how [BalancerV2](https://github.com/balancer-labs/balancer-v2-monorepo/blob/master/pkg/pool-utils/contracts/BasePool.sol#L307-L325) and [UniswapV2](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L121) do it. some MINIMUM amount of pool tokens get burnt when the first mint happens
-  You can also go [BentoBox](https://github.com/sushiswap/bentobox/blob/canary/contracts/BentoBox.sol) route of allowing total supply to go zero but not anywhere between 0 and MINIMUM. 
