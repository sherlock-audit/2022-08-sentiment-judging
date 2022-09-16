0xNazgul
# [NAZ-M5] No Storage Gap for Upgradeable Contract Might Lead to Storage Slot Collision

## Summary
There are no storage gaps used in the upgradeable contracts. This could lead to future issues such as storage slot collision.

## Vulnerability Detail
For upgradeable contracts, there must be storage gap to "allow developers to freely add new state variables in the future without compromising the storage compatibility with existing deployments" (quote OpenZeppelin). Otherwise it may be very difficult to write new implementation code. Without storage gap, the variable in child contract might be overwritten by the upgraded base contract if new variables are added to the base contract. This could have unintended and very serious consequences to the child contracts, potentially causing loss of user fund or cause the contract to malfunction completely.

Refer to the bottom part of this [`article`](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable).

## Impact
These contracts don't contain a storage gap. The storage gap is essential for upgradeable contracts because "It allows us to freely add new state variables in the future without compromising the storage compatibility with existing deployments". Refer to the bottom part of this [`article`](https://docs.openzeppelin.com/contracts/3.x/upgradeable)

If the contracts inheriting these contracts contains additional variables, then the base contract cannot be upgraded to include any additional variable, because it would overwrite the variable declared in its child contract. This greatly limits contract upgradeability.

## Code Snippet
[`Account.sol`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/Account.sol), [`AccountManager.sol`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/AccountManager.sol), [`RiskEngine.sol`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/RiskEngine.sol), [`LToken.sol`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/tokens/LToken.sol)

## Tool used
Manual Review

## Recommendation
Recommend adding appropriate storage gap at the end of upgradeable contracts such as the below. Please reference OpenZeppelin upgradeable contract templates.
```js
uint256[50] private __gap;
```