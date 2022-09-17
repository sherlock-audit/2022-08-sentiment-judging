pashov
# `LToken.sol`'s `collectFrom()` method does not actually move any value

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LToken.sol#L153](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LToken.sol#L153)

### Proof of concept

The code in `collectFrom()` method updates the accounting logic of the contract but forgets to actually transfer tokens from the given account back to itself.

### Impact

Not being able to collect assets after lending them (and actually transferring assets when lending them) will result in permanent loss of funds for the protocol.

### Recommendation

In `collectFrom()` add the following code

```jsx
asset.safeTransferFrom(account, address(this), amt);
```