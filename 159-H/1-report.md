pashov
# `ERC20.sol` is susceptible to classic ERC20 `approve` functionality front-running exploit

### Scope

[https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/utils/ERC20.sol#L9](https://github.com/sherlock-audit/2022-08-sentiment-pashov/blob/d87a56ad4e2713682064b3bd876477b27d44f3a2/protocol/src/tokens/utils/ERC20.sol#L9)

### Proof of concept

The problem is perfectly described here [https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit)

The tldr; is that if you approved a person to spend 100 tokens and then you want to decrease his allowance to 50, if he spends his 100 tokens allowance before you set his allowance to 50 he will be able to spend 50 more. This results in him being able to spend 150 tokens instead of your desired 50 allowance.

### Impact

This can lead to much more token amount being transferred from a user’s wallet than what he wants, since if he is decreasing the allowance he would want less tokens to be spent from his wallet, but this way actually more tokens could be spent.

### Recommendation

Add OpenZeppelin’s [increaseAllowance](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3dac7bbed7b4c0dbf504180c33e8ed8e350b93eb/contracts/token/ERC20/ERC20.sol#L181) and [decreaseAllowance](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3dac7bbed7b4c0dbf504180c33e8ed8e350b93eb/contracts/token/ERC20/ERC20.sol#L201) functionality to `ERC20.sol`.