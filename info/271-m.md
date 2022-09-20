Bahurum
# Lack of efficient failsafe mechanism in `AccountManager`

## Summary
Pausing/unpausing does not put funds at risk, but can prevent losses in case of exploits or high volatility. The current admin (multisig or goverance) controls pausing/unpausing but doesn't have the reaction time necessary to pause the contract in time.

## Vulnerability Detail
The protocol has many external interactions and uses different oracles, giving a large attack surface. The main concern is unlimited borrowing through oracle manipulation. While the cross-margin lending is a feature of the protocol, it puts all the lending pools at risk when one oracle price fails or is manipulated. Even if risks are reduced with security audits and proper maintainance and monitoring, the dependency on oracles makes the probability of an exploit non negligible, while the funds at risk are the entire protocol funds.   
For this a fail safe mechanism is needed. `AccountManager` has a pausing functionality that stops bowworing, but it is controlled by the same address (`admin`) as `initDep()`. `initDep()` is a critical function and is controlled either by multisig or governance. It takes many hours to put together a multisig, days for a governance vote, while `pause()` is supposed to be used as an emergency switch, but it cannot be useful in case of rapid market movements (flash crash) or oracle failure/manipulation since the delay to call it would be too long.  
There is another issue: pausing the contract does not prevent an attacker to steal funds in some cases(`withdraw()` and `withdrawEth()` don't have the `whenNotPaused` modifier). For example 

0. Attacker deposits collateral
1. Attacker borrows token A 
2. Attacker Somehow manipulates the oracle price of token A (lower than market price more than approx. 20%)
3. Attacker withdraws more than initial collateral.


## Impact
No way to prevent theft of funds in time if an oracle manipulation or extreme market movement occurs. 

## Code Snippet
https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L68-L81

```solidity
    function init(IRegistry _registry) external {
        if (initialized) revert Errors.ContractAlreadyInitialized();
        initialized = true;
        initPausable(msg.sender); // @audit msg.sender is set as admin
        registry = _registry;
    }

    /// @notice Initializes external dependencies
    function initDep() external adminOnly {
        riskEngine = IRiskEngine(registry.getAddress('RISK_ENGINE'));
        controller = IControllerFacade(registry.getAddress('CONTROLLER'));
        accountFactory =
            IAccountFactory(registry.getAddress('ACCOUNT_FACTORY'));
    }
```

## Tool used

Manual Review

## Recommendation
`admin` should maintain the ability to pause/unpause `AccountManager`. 
Define also a `pauser` address which can be set and changed by `admin`.
The `pauser` should be able to pause borrowing too.
The `pauser` should be a monosig (if possible a bot that monitors the markets/oracles).  
`pauser` should also be able to pause `withdraw()` and `withdrawEth()`. The concern about users funds being locked are not valid because the admin can disable withdrawals anyway by changing the oracle by calling `RiskEngine.initDep()`.