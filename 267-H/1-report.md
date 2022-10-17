WATCHPUG
# Tokens received from Curve's `remove_liquidity()` should be added to the assets list even if `_min_amounts` are set to `0`

## Summary

Curve controller's `canRemoveLiquidity()` should return all the underlying tokens as `tokensIn` rather than only the tokens with `minAmount > 0`.

## Vulnerability Detail

https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/curve/StableSwap2PoolController.sol#L129-L152

```solidity
function canRemoveLiquidity(address target, bytes calldata data)
    internal
    view
    returns (bool, address[] memory, address[] memory)
{
    (,uint256[2] memory amounts) = abi.decode(
        data[4:],
        (uint256, uint256[2])
    );

    address[] memory tokensOut = new address[](1);
    tokensOut[0] = target;

    uint i; uint j;
    address[] memory tokensIn = new address[](2);
    while(i < 2) {
        if(amounts[i] > 0)
            tokensIn[j++] = IStableSwapPool(target).coins(i);
        unchecked { ++i; }
    }
    assembly { mstore(tokensIn, j) }

    return (true, tokensIn, tokensOut);
}
```

The `amounts` in Curve controller's `canRemoveLiquidity()` represent the "Minimum amounts of underlying coins to receive", which is used for slippage control.

At L144-149, only the tokens that specified a minAmount > 0 will be added to the `tokensIn` list, which will later be added to the account's assets list.

We believe this is wrong as regardless of the minAmount `remove_liquidity()` will always receive all the underlying tokens.

Therefore, it should not check and only add the token when it's minAmount > 0.

## Impact

When the user set `_min_amounts` = `0` while removing liquidity from `Curve` and the withdrawn tokens are not in the account's assets list already, the user may get liquidated sooner than expected as `RiskEngine.sol#_getBalance()` only counts in the assets in the assets list.

## Code Snippet

https://arbiscan.io/address/0x7f90122bf0700f9e7e1f688fe926940e8839f353#code

```vyper
@external
@nonreentrant('lock')
def remove_liquidity(
    _burn_amount: uint256,
    _min_amounts: uint256[N_COINS],
    _receiver: address = msg.sender
) -> uint256[N_COINS]:
    """
    @notice Withdraw coins from the pool
    @dev Withdrawal amounts are based on current deposit ratios
    @param _burn_amount Quantity of LP tokens to burn in the withdrawal
    @param _min_amounts Minimum amounts of underlying coins to receive
    @param _receiver Address that receives the withdrawn coins
    @return List of amounts of coins that were withdrawn
    """
    total_supply: uint256 = self.totalSupply
    amounts: uint256[N_COINS] = empty(uint256[N_COINS])

    for i in range(N_COINS):
        old_balance: uint256 = self.balances[i]
        value: uint256 = old_balance * _burn_amount / total_supply
        assert value >= _min_amounts[i], "Withdrawal resulted in fewer coins than expected"
        self.balances[i] = old_balance - value
        amounts[i] = value
        ERC20(self.coins[i]).transfer(_receiver, value)

    total_supply -= _burn_amount
    self.balanceOf[msg.sender] -= _burn_amount
    self.totalSupply = total_supply
    log Transfer(msg.sender, ZERO_ADDRESS, _burn_amount)

    log RemoveLiquidity(msg.sender, amounts, empty(uint256[N_COINS]), total_supply)

    return amounts
```

## Tool used

Manual Review

## Recommendation

`canRemoveLiquidity()` can be changed to:

```solidity
function canRemoveLiquidity(address target, bytes calldata data)
    internal
    view
    returns (bool, address[] memory, address[] memory)
{
    address[] memory tokensOut = new address[](1);
    tokensOut[0] = target;

    address[] memory tokensIn = new address[](2);
    tokensIn[0] = IStableSwapPool(target).coins(0);
    tokensIn[1] = IStableSwapPool(target).coins(1);
    return (true, tokensIn, tokensOut);
}
```
## Sentiment Team
Fixed as recommended. PR [here](https://github.com/sentimentxyz/controller/pull/39).

## Lead Senior Watson
Confirmed fix. 
