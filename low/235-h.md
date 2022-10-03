Waze
# Address validation is required when using safeTransferETH.

## Summary
zero address in solidity has a special consideration. zero address become significant because state variables or local variables have zero value by default. zero addess used as burn adresses because the private key corresponding to this zero adress is not known so when the token sent to this adress can made token inaccessable. safeTransferETH() not use zero check validation it lead to ETH lost when transfer to zero address.
## Vulnerability Detail
when the sender send any token with safeTransferETH and the sender made mistakes by not recheck the address to send ETH . it makes all ETH will be lost or burn due to safeTransferETH() not use zero address validation.
## Impact
the token will be lost when the destiation address is zero address
## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-calmkidd/blob/5d8b0126f4e4dcdab99c5c89922d74faa99cecb3/protocol/src/core/AccountManager.sol#L135
https://github.com/sherlock-audit/2022-08-sentiment-calmkidd/blob/5d8b0126f4e4dcdab99c5c89922d74faa99cecb3/protocol/src/core/Account.sol#L173
## Tool used

Manual Review

## Recommendation
recheck the destination address when using safeTransferETH.