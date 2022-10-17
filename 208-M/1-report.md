GalloDaSballo
# M-05 Yearn `withdraw(uint)` selector may backfire

## Summary

The [YearnController](https://github.com/sentimentxyz/controller/blob/a2ddbcc00f361f733352d9c51457b4ebb999c8ae/src/yearn/YearnController.sol#L24-L25) is using:

```
   /// @notice withdraw(uint256) function signature
    bytes4 constant WITHDRAW = 0x2e1a7d4d;
```

Which is the selector for withdraw(amount, msg.sender, 1) where the max_loss is 1bps

See Yearn Code: https://github.com/yearn/yearn-vaults/blob/efb47d8a84fcb13ceebd3ceb11b126b323bcc05d/contracts/Vault.vy#L1010

## Vulnerability Detail

On one hand the selector will allow for loss, if you'd want to have no loss you should change it to a function call with a hardcoded 0 value for `maxLoss`

On the other hand, if the account has too big of a position in a Yearn Vault (e.g. more than 10% of the vault), then a withdrawal may simply not be possible as the function will always revert.

You can monitor the limits on `https://yearn.watch/`, per the Yearn [`withdraw`](https://github.com/yearn/yearn-vaults/blob/efb47d8a84fcb13ceebd3ceb11b126b323bcc05d/contracts/Vault.vy#L1084) code, the function will go through the withdrawal queue, adding up losses (if any) to the caller.

Because the hardcoded selector will not offer any flexibility to the `maxLoss` parameter, in any scenario in which the losses are above 1 bps, the function will simply revert, preventing any withdrawal.

## Tool used

Manual Review

## Recommendation

Consider adding support for a more complete withdraw that allows to change the `maxLoss` parameter

## Sentiment Team
Not fixing since we don't plan to launch with Yearn. We are okay with not including this contract in the coverage scope since it won't be deployed.

## Lead Senior Watson
Acknowledged.
