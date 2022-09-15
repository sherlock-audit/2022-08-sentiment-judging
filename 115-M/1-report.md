rajatbeladiya
# accountsOwnedBy() will run out of gas.

## Summary
transaction will be reverted if `accounts` array will be large enough.

## Vulnerability Detail
addAccount() pushes the account address to `accounts` array. If accounts array will be large enough, `accountsOwnedBy()` will not work because it looping over accounts and it will be reverted because of transaction run out of gas.

## Impact
`accountsOwnedBy()` function will not work and not able to get accounts owner by individual owner.

## Code Snippet

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Registry.sol#L183-L188


## Tool used
Manual Review

## Recommendation
remove account onClose() account or revise it
