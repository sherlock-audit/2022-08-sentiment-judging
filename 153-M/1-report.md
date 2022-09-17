ellahi
# No support for Fee-on-Transfer / Rebase tokens.

## Summary
No support for Fee-on-Transfer / Rebase tokens. 
According to one of the devs on discord, it seems like FoT / rebasing tokens aren't completely out of the picture.
> as of today, there are no plans to integrate with any fee-on-transfer / rebasing tokens as a day1 feature and accordingly we haven't made any special provisions around the same.
But we'd def want to allow those in the future.
## Vulnerability Detail
Lending and borrowing accounting is done without taking potential fees into consideration. This would cause bad accounting and unintended consequences.
## Impact
Medium
## Code Snippet
[`LToken.sol`](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LToken.sol#L128-L165)
## Tool used
Manual Review
## Recommendation
If planning to add FoT / Rebase tokens, consider changing how the accounting is done to support such tokens.