kirk-baird
# HIGH: Malicious Assets Can Be Added By Calling `removeLiquidity()` in SushiSwap

## Summary

It's possible to add an arbitrary contracts to the list in `Account.assets` by calling `removeLiquidity()` on SushiSwap. These contracts  have not been approved for use by the protocol.

## Vulnerability Detail

`UniV2Controller.sol` which is used by SushiSwap does not validate `tokenIn` for `removeLiquididty()` and `removeLiquidityEth()`.

We can therefore add any arbitrary asset to `Account.assets`.

Steps:
a) `UniswapFactoryV2.createPair()` for two malicious tokens
b) The attacker than deposits some liquidity to get some LP tokens for this pair
c) These LP tokens can be transferred to the attackers `Account`
d) Call `AccountManager.exec()` via the sushi swap router to redeem these LP tokens. This will have `tokensIn` as our two malicious tokens. They will be added to `Account.asset`.

## Impact

The impact of adding arbitrary tokens to `Account.assets` is that these will not be set in the oracle. Hence `oracle.getPrice(token)` will revert.
Since the price is required in `RiskEngine.sol` to calculate the health position. If this reverts it will be impossible for that account to be liquidated even if the position is unhealthy.
Note the attacker can still call `settle()` after which they may withdraw other tokens.

An attacker may take out a risky position by borrowing then use the steps mentioned above to add malicious contracts to the `Account.assets` list.
The attacker waits for the position to change, if the position improves they repay their debts at a cheaper price (`repay()` does not call  `oracle.getPrice(token)` for the malicious tokens).
If the position deteriorates the attacker does not repay their debts and cannot be liquidated causing the lenders to incur the loss.

Preventing liquidations will impact the overall safety guarantees of the system and lenders may be unable to ever reclaim the lent funds.

## Code Snippet

```solidity
    /**
        @notice Evaluates whether liquidity can be removed
        @param data calldata for removing liquidity
        @return canCall Specifies if the interaction is accepted
        @return tokensIn List of tokens that the account will receive after the
        interactions
        @return tokensOut List of tokens that will be removed from the account
        after the interaction
    */
    function removeLiquidity(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address tokenA, address tokenB) = abi.decode(data, (address, address));

        address[] memory tokensIn = new address[](2);
        tokensIn[0] = tokenA;
        tokensIn[1] = tokenB;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);

        return(true, tokensIn, tokensOut);
    }

    /**
        @notice Evaluates whether liquidity can be removed
        @param data calldata for removing liquidity
        @return canCall Specifies if the interaction is accepted
        @return tokensIn List of tokens that the account will receive after the
        interactions
        @return tokensOut List of tokens that will be removed from the account
        after the interaction
    */
    function removeLiquidityEth(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address token) = abi.decode(data, (address));

        address[] memory tokensIn = new address[](1);
        tokensIn[0] = token;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(token, WETH);

        return(true, tokensIn, tokensOut);
    }
```

## Tool used

Manual Review

## Recommendation

The following checks should be added `controller.isTokenAllowed(tokensIn[0])` and `controller.isTokenAllowed(tokensIn[1])` to both `removeLiquidity()` and `removeLiquidityEth()`. 

This will prevent redeeming LP tokens for malicious tokens. Note that `AccountManager.withdraw()` can still be called to extract untrusted LP tokens if the `Account` has no debt.
