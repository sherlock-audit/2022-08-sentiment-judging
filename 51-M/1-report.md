grhkm
# Frontrunning in the Contract's Initialization Function Can Lead to an Array of Vulnerabilities

## Summary
There exists an issue in the `Account.sol`, `AccountManager.sol` and the `Registry.sol` contracts where front running allows the attacker to cause adverse effects on the contract should they succeed. 

## Vulnerability Analysis
Given the `init()` function in the `Registry.sol` contract:
```solidity
    /**
        @notice Contract Initialization function
        @dev Can only be invoked once
    */
    function init() external {
        if (initialized) revert Errors.ContractAlreadyInitialized();
        initialized = true;
        initOwnable(msg.sender);
    }
```
In the above snippet of code, the `init()` function is used to determine ownership of the contract in question (with varying state changes depending on the contract). An attacker can monitor the mempool for a contract deployment which takes a certain amount of time depending on the congestion of transactions waiting to be processed. Should an adversary catch hold of the deployment transaction, they are free to call the `init()` function to gain ownership of the contract.

I have awarded this as a "Medium" in severity because certain conditions must be met in order to execute a successful attack. The attacker should actively be monitoring the mempool for opportunities in order to exploit this attack vector.

## Impact
The following contracts are affected with the described impact:
- `Account.sol`:`59`
	- Attackers can initialize a malicious `accountManager` to manipulate funds or brick the contract.
- `AccountManager.sol`:`68`
	- `initPausable()` is called here which may be able to brick the contract if `pause()` was called by an unwanted address.
- `Registry.sol`:`61`
	- Complete ownership over the registry and user accounts as the administrator. 

## Proof of Concept
#### Exploit scenario in the case of `Registry.sol`
- Alice deploys the `Registry.sol` contract to the blockchain.
- Eve notices the contract has successfully been deployed and sees a front running opportunity by calling the `init()` function to become the admin user. 
- Eve is free to use whichever functions in the deployed contract which require an administrative privileges.  

I have included a proof of concept test outlining the above scenario as followed:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import {Errors} from "../../utils/Errors.sol";
import {TestBase} from "../utils/TestBase.sol";
import {IRegistry} from "../../interface/core/IRegistry.sol";
import {Registry} from "../../core/Registry.sol";

// Located in protocol/src/test/MyTests/TestFrontRun.sol

contract TestFrontRun is TestBase {

    address alice;
    address eve;

    function setUp() public {

        alice = cheats.addr(1);
        eve = cheats.addr(2);

    }

    function testProveFrontRunning() public {
        
        cheats.startPrank(alice);
        // Alice, the rightful owner deploys the registry contract
        Registry myReg = new Registry();
        cheats.stopPrank();

        assertEq(address(myReg.admin()), address(0));

        // Eve, a malicious user is monitoring the mempool and can frontrun alice to initialize the contract
        cheats.startPrank(eve);
        myReg.init();
        cheats.stopPrank();

        assertEq(address(myReg.admin()), address(eve));

    }

}
```

## Tool used
Manual Review

## Recommendation
Consider the following remediations to prevent a frontrunning attack on the deployed contracts:
- Implement access controls in the form of a modifer so only the deployer can call the initialization functions. 
- Consider moving the initialization operations to a contract constructor for the purpose of executing initialization code on deployment. 
