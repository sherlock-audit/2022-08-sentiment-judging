cergyk
# Lack of target address validation in function exec in accountManager.sol can let malicious account owner execute instructions to insert malicious tokensIn and tokensOut 

## Summary

Lack of target address validation in function exec in accountManager.sol can let malicious account owner execute any instructions

## Vulnerability Detail

In accountManager.sol, we have a function exec

```
   /**
        @notice A general function that allows the owner to perform specific interactions
            with external protocols for their account
        @dev Target must have a controller in controller facade
        @param account Address of account
        @param target Address of contract to transact with
        @param amt Amount of Eth to send to the target contract
        @param data Encoded sig + params of the function to transact with in the
            target contract
    */
    function exec(
        address account,
        address target,
        uint amt,
        bytes calldata data
    )
        external
        onlyOwner(account)
    {
        bool isAllowed;
        address[] memory tokensIn;
        address[] memory tokensOut;
        (isAllowed, tokensIn, tokensOut) =
            controller.canCall(target, (amt > 0), data);
        if (!isAllowed) revert Errors.FunctionCallRestricted();
        _updateTokensIn(account, tokensIn);
        (bool success,) = IAccount(account).exec(target, amt, data);
        if (!success)
            revert Errors.AccountInteractionFailure(account, target, amt, data);
        _updateTokensOut(account, tokensOut);
        if (!riskEngine.isAccountHealthy(account))
            revert Errors.RiskThresholdBreached();
    }
```

according to the comment,

```
 @dev Target must have a controller in controller facade
```

but there is no validation implemented for the target address, the target address and the call data may be malicious.

first, the controller.canCall check can by passed with a forged call data.

```
(isAllowed, tokensIn, tokensOut) =
            controller.canCall(target, (amt > 0), data);
```

for example, if the controller is AaveEthController.sol

```
    /// @notice depositETH(address,address,uint16) function signature
    bytes4 public constant DEPOSIT = 0x474cf53d;

    /// @notice withdrawETH(address,uint256,address) function signature
    bytes4 public constant WITHDRAW = 0x80500d20;
```

```
    /// @inheritdoc IController
    function canCall(address, bool, bytes calldata data)
        external
        view
        returns (bool, address[] memory, address[] memory)
    {
        bytes4 sig = bytes4(data);
        if (sig == DEPOSIT) return (true, tokens, new address[](0));
        if (sig == WITHDRAW) return (true, new address[](0), tokens);
        return (false, new address[](0), new address[](0));
    }
```

to be pass the check,

if the target address is malicious, the hacker can just create a function that has the same function signature hash

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.2;

contract POC {

    function depositETH(address a,uint256 b,address c) public {
       // malicious instructions.
    }

}
```

the forged call data is 

0x474cf53d + rest of call data (encoded function arugment instructions).

0x474cf53d is "deposit".

it has been historically prove that using function signature as validation mechanism is not reliable.

One of big failure is the poly network hack, resulting in the loss of billions. The step of the hack is forging function signature as call data.

https://rekt.news/polynetwork-rekt/

then the function execute the instruction

```
(bool success,) = IAccount(account).exec(target, amt, data);
```

and the exec function in account.sol is 

```
    function exec(address target, uint amt, bytes calldata data)
        external
        accountManagerOnly
        returns (bool, bytes memory)
    {
        (bool success, bytes memory retData) = target.call{value: amt}(data);
        return (success, retData);
    }
```

in the line,

```
 (bool success, bytes memory retData) = target.call{value: amt}(data);
```

assume the target and the call data is malicious, the bad instruction is executed in this line of code.

there is another attack vector.

check the UniV2Controller.sol

```
    function removeLiquidity(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address tokenA, address tokenB) = abi.decode(data, (address, address));

        address[] memory tokensIn = new address[](2);
        tokensIn[0] = tokenA;
        tokensIn[1] = tokenB;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);

        return(true, tokensIn, tokensOut);
    }
```

the hacker can controll the token A and token B passed in the line below

```
(address tokenA, address tokenB) = abi.decode(data, (address, address));
```

then tokensIn is messed up when calling

```
 _updateTokensIn(account, tokensIn);
```
malicious user can insert bad token 

mint a large amount of tokensIn to mess up with the account balance.

the malicious user can create a bad token pool, with token pair tokenA and tokenB. token A and token B are both set by hackers.

then in

```
  tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);
```

tokensOut is polluted.

the user can trick the program to call

```
 _updateTokensOut(account, tokensOut);
```

and remove the "tokensOut" token.

## Tool used

Manual Review

Foundry

## Recommendation

We recommend the protocol verify the target address and whitelist the target address to make sure the above attack won't happen.