pashov
# Possible stuck user Ether in `LEther.sol`

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LEther.sol#L55](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/LEther.sol#L55)

### Proof of concept

The `LEther.sol` contract is used to deposit Ether (`depositEth()` is `payable`) into it and get proportional shares or vice-versa.  The problem is that the contract has a `receive()` function that will be called if a user’s EOA or a smart contract calls the `LEther.sol` contract with empty calldata, which can happen by mistake.

Example scenario is a user wants to call `depositEth()` with msg.value of 10 ETH, but he  sends empty calldata instead by mistake. Since `LEther.sol` does not have a mechanism to withdraw stuck Ether, all of the user funds sent will be stuck in the smart contract.

### Impact

The impact is permanent loss of user funds due to incorrect call (empty calldata) of the `LEther.sol` smart contract

### Recommendation

Remove the `receive()` function from `LEther.sol`