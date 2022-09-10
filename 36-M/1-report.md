cergyk
#  Contract Locking Received Ethers 

## Summary
Contract locking ether that comes into 

## Vulnerability Detail
Receive function is a payable and anyone can deposit ether to contract but function do not doing any operation with deposited ether. 


## Impact
Ether locks for ever in the contract Account.sol

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/Account.sol#L196
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/LEther.sol#L55


## Tool used

Manual Review

## Recommendation
If the function has nothing to do with money, it should revert it or remove payable.

```receive() public payable { revert (); }```
When the user tries to send eth to contract, receive() do revert eth to the user.

```receive() public {}```
When receive function doesn't have a payable attribute, the user can't send eth to contract.


## Reference:
https://github.com/crytic/slither/wiki/Detector-Documentation#contracts-that-lock-ether

