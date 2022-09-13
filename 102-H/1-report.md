0xSmartContract
# Storage Write Removal Bug On Conditional Early Termination

## Summary
On September 5, 2022, a bug in Solidity’s Yul optimizer was found by differential fuzzing.

The bug was [Solidity version 0.8.17](https://github.com/ethereum/solidity/releases/tag/v0.8.17), released on September 08, 2022, provides a fix. The bug is significantly easier to trigger with optimized via-IR code generation, but can theoretically also occur in optimized legacy code generation.

The bug may result in storage writes being incorrectly considered redundant and removed by the optimizer. The problem manifests in presence of assembly functions that may conditionally terminate the external EVM call using the return() or stop() opcode.

Soliditylang assigned the bug a severity of “medium/high”.
https://blog.soliditylang.org/2022/09/08/storage-write-removal-before-conditional-termination/

## Vulnerability Detail

Similar to the problem mentioned above, there is a ```return``` in the following inline assembly block in the project.

https://github.com/sherlock-audit/2022-08-sentiment-0xSmartContract/blob/main/protocol/src/proxy/BaseProxy.sol#L35-L53

```js
function _delegate(address impl) internal virtual {
        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())

            let result := delegatecall(gas(), impl, ptr, calldatasize(), 0, 0)

            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size)
            }
        }
    }

```

## Impact

## Code Snippet
[BaseProxy.sol#L35-L53](https://github.com/sherlock-audit/2022-08-sentiment-0xSmartContract/blob/main/protocol/src/proxy/BaseProxy.sol#L35-L53)

```js
function _delegate(address impl) internal virtual {
        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())

            let result := delegatecall(gas(), impl, ptr, calldatasize(), 0, 0)

            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size)
            }
        }
    }

```

## Proof of Concept

The ```_delegate``` function return according to the ```impl address```  argument, while returning a normal address, it is expected to revert if the address is ```0```, but it can be seen in the process that it returns at address ```0```

```js

contract Test is DSTest {
  
    Contract0 c0;
    Contract1 c1;
    
    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
    }
    
    function test() public {
        address testadd = address(0) ;
        c0._delegate(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4);
        c1._delegate(testadd);

    }
}

contract Contract0 {

function _delegate(address impl) public virtual {
        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())

            let result := delegatecall(gas(), impl, ptr, calldatasize(), 0, 0)

            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size) 
            }
        }
    }
}



contract Contract1 {

function _delegate(address impl) public virtual {
        assembly {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())

            let result := delegatecall(gas(), impl, ptr, calldatasize(), 0, 0)

            let size := returndatasize()
            returndatacopy(ptr, 0, size)

            switch result
            case 0 {
                revert(ptr, size)
            }
            default {
                return(ptr, size) 
            }
        }
    }

}


Foundry Test Traces:

  [176245] Test::setUp() 
    ├─ [42293] → new Contract0@"0xce71…c246"
    │   └─ ← 211 bytes of code
    ├─ [42293] → new Contract1@"0x185a…1aea"
    │   └─ ← 211 bytes of code
    └─ ← ()

  [15918] Test::test() 
    ├─ [2914] Contract0::_delegate(0x5b38da6a701c568545dcfcb03fcb875f56beddc4) 
    │   ├─ [0] 0x5b38…ddc4::_delegate(0x5b38da6a701c568545dcfcb03fcb875f56beddc4) [delegatecall]
    │   │   └─ ← ()
    │   └─ ← ()
    ├─ [2914] Contract1::_delegate(0x0000000000000000000000000000000000000000) 
    │   ├─ [0] 0x0000…0000::_delegate(0x0000000000000000000000000000000000000000) [delegatecall]
    │   │   └─ ← ()
    │   └─ ← ()
    └─ ← ()



```



## Tool used

Manual Review

## Recommendation
Use Solidity version 0.8.17    (https://github.com/ethereum/solidity/releases/tag/v0.8.17)

