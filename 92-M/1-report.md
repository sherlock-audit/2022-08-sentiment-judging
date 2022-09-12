PwnPatrol
# Uniswap contract added to controller doesn't match with function signatures

## Summary

In the `ArbiDeploymentFlow.md` doc, it specifies the plan to add the UniV3Controller to controllerFacade, and then update it so it applies to Router (address: 0xE592427A0AEce92De3Edee1F18E0157C05861564). 

However, some of the function signatures of the Router at this address don't match the signatures specified in the controller, so function calls will revert.

## Vulnerability Detail

In `UniV3Controller.sol`, the function signatures that are allowed to be used when calling the contract are specified.

However, these signatures differ between the two Uniswap V3 router contracts due to a change in the parameters for the arguments (in all four functions, the original router included a uint256 for duration, and the Router 2 contract does not).

As a result, the function signatures in the controller do not line up with the signatures in the Router contract that is specified in the deployment doc:
- exactInputSingle should be 0x414bf389, not 0x04e45aaf
- exactOutputSingle should be 0xdb3e2198, not 0x5023b4df
- exactInput should be 0xc04b8d59, not 0xb858183f
- exactOutput should be 0xf28c0498, not 0x09b81346

## Impact

Users attempting to interact with Uniswap V3 through the UniV3Controller will have their calls rejected due to mismatched function signatures.

## Code Snippet

```solidity
    /// @notice exactInputSingle((address,address,uint24,address,uint256,uint256,uint160)) function signature
    bytes4 constant EXACT_INPUT_SINGLE = 0x04e45aaf;

    /// @notice exactOutputSingle((address,address,uint24,address,uint256,uint256,uint160))	function signature
    bytes4 constant EXACT_OUTPUT_SINGLE = 0x5023b4df;

    /// @notice exactInput((bytes,address,uint256,uint256)) function signature
    bytes4 constant EXACT_INPUT = 0xb858183f;

    /// @notice exactOutput((bytes,address,uint256,uint256)) function signature
    bytes4 constant EXACT_OUTPUT = 0x09b81346;
```

## Tool Used

Manual Review

## Recommendation

Use Router 2 (address: 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45) when calling `controllerFacade.updateController()` to connect UniV3Controller to the correct router.

Alternatively, add the extra function signatures in `UniV3Controller.sol` so the controller is able to work on either Router.