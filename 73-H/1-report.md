cergyk
# LEther token has payable receive() function that doesn't check anything 

## Summary

`LEther` token has payable `receive()` function that doesn't check the sender and doesn't revert.

## Vulnerability Detail

`LEther` token has payable `receive()` function that is empty. Also `LEther` has `depositEth()` payable function and this vault suppose to get ether from the sender. User can by error send ethers directly to the `LEther`. Contract should check this and in case if the money were sent not by specific address then revert or call `depositEth()` and mint some tokens to the senders. Currently sender will lost funds.

## Impact

The sender will lose his funds.

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-rvierdiyev/blob/main/protocol/src/tokens/LEther.sol#L55

## Tool used

Manual Review

## Recommendation
If you need to have the ability to top up `LEther`, then make it to be `Ownable` or provide some specific address that can top up that contract. And check for it in `receive()`, otherwise  - revert or call `depositEth()`.