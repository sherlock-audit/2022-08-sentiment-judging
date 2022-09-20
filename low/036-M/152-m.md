ellahi
# No way to recover ETH sent to `LEther.sol` via `receive`.

## Summary
There is no way to recover ETH that has been sent to `LEther.sol` via the `receive` function.
## Vulnerability Detail
The contract `LEther.sol` contains a `receive` function which allows the contract to receive ETH. If a user mistankingly sends over some ETH without invoking `depositEth` expecting to receive wrapped eth back, they will receive nothing and their ether is stuck. 
## Impact
Medium.
## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LEther.sol#L55-L55

```solidity
receive() external payable {}
```
## Tool used
Manual Review

## Recommendation
Consider removing the `receive` function or calling the `depositEth` function inside the `receive` function.
