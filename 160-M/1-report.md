pashov
# User can mistakenly mint his shares to the zero address

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/utils/ERC20.sol#L183](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/utils/ERC20.sol#L183)

### Proof of concept

The custom implementation of ERC20 `ERC20.sol` does not check if the `to` argument value in the `_mint()` function is not the zero address. This missing check bubbles up in the inheritance chain from `ERC20.sol` to `ERC4626.sol` to `LToken.sol`. When a user wants to mint shares of an LToken he will call the `mint()` function coming from `ERC4626.sol` taking an `address receiver` argument, which again is not checked if it is not the zero address. So basically, if a user calls an LToken’s `mint()` function with a `receiver` argument value of the zero address by mistake, the transaction will execute successfully but the user won’t receive any shares.

### Impact

This vulnerability can result in a loss of funds for a user of the protocol, because all of his funds deposited will be transferred to shares that go to the zero address, so they can’t be redeemed/withdrawn to the original amount.