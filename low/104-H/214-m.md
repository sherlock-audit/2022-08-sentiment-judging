Chom
# Account healthy can't be checked if borrowing or supplying too many assets

## Summary
Account healthy can't be checked if borrowing or supplying too many assets

## Vulnerability Detail
isAccountHealthy calls _getBalance and _getBorrows which will revert with the gas limit if too many assets are borrowed or supplied.

## Impact
- Can't liquidate -> Cause bad debt
- Can't borrow

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/protocol/src/core/RiskEngine.sol#L121-L126

## Tool used

Manual Review

## Recommendation

Limit the number of available assets
