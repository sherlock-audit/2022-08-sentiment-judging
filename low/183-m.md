PwnPatrol
# Token updates/migrations pose an insolvency risk to the system and can break the Liquidation Flow

## Summary
Token updates/migrations pose an insolvency risk to the system and can break the Liquidation Flow

## Vulnerability Detail
It's not rare for some tokens to get updated/migrated to new version (example OHM). What is more, oracles deprecate data feeds for old token versions.

If such event happens, it will break Liquidation and Settlement Flows for anyone holding old version tokens.

The probability of such an event is low and cannot be triggered by Attacker. On the other hand, the severity of such an event is high thus I'll mark this as Medium. 

## Impact
Protocol Insolvency Risk.
Liquidation Flow DoS.

## Code Snippet
```solidity
function _valueInWei(address token, uint amt)
    internal
    view
    returns (uint)
{
    return oracle.getPrice(token) // reverts here for migrated tokens
    .mulDivDown(
        amt,
        10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
    );
}
```

## Tool used

Manual Review

## Recommendation
Our recommendation is to implement a list with tokens which will be skipped/filtered during Liquidation Flow and Getitng Balance Flow.