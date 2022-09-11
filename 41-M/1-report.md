grhkm
# Native tokens can be locked forever in the contract LEther

## Summary
Native tokens can be locked forever in the contract LEther.

## Vulnerability Detail
The contract LEther allows users to deposit their native tokens and get in exchange shares. To do so, they should use the function depositEth().
However, there is also a function receive() that accepts native tokens. Due to the purpose of the contract LEther, users could be misled and use receive() instead of depositETH(). As it is not possible to withdraw native tokens from the contract LEther, funds would be locked forever.

## Impact

Funds of misled users (native tokens) can be locked forever in the contract LEther.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-validydy/blob/2123357e2a9866bd62d8fe731b222f917a062d59/protocol/src/tokens/LEther.sol#L55

## Tool used

Manual Review

## Recommendation

Three ideas:
- remove the function receive() if it is not used.
- or add an error "DirectNativeTransfersNotAllowed".
- or add a function withdrawNativeTokens(), that can be called only by an admin address.
