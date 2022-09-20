HonorLt
# Missing validations

## Summary
There are at least a few validations that could be enforced to make the system more robust. I am grouping them into one submission because I feel they are quite important but might be too small for separate submissions.

## Vulnerability Detail
1) ```LEther``` contract contains an empty fallback function with no way to retrieve it back:
```solidity
  receive() external payable {}
```
Probably meant to receive ETH from the WETH contract, thus can add ```require msg.sender == asset``` to prevent accidental loss of ETH.


2) ```ERC4626``` contract does not explicitly implement ```IERC4626```:
```solidity
  abstract contract ERC4626 is CustomERC20
```
Consider adding ```is IERC4626``` to enforce compile time checks.


3) Anyone can open an account on behalf of an owner:
```solidity
  function openAccount(address owner) external whenNotPaused
```
This may result in a poor user experience, e.g. if a frontend displays all the accounts for that user. Consider adding restrictions, e.g. only the owner or an approved address can perform such actions.


4) Chainlink oracles do not check if the data is not stale:
```solidity
  (, int answer,,,) =
      feed[token].latestRoundData();

  if (answer < 0)
      revert Errors.NegativePrice(token, address(feed[token]));
```

From Chainlink's documentation:

**if answeredInRound < roundId could indicate stale data.**

**A timestamp with zero value means the round is not complete and should not be used.**

Consider adding extra checks for stale data:
```solidity
(
    uint80 roundID,
    int256 answer,
    ,
    uint256 timestamp,
    uint80 answeredInRound
) = feed[token].latestRoundData();

if (answer <= 0)
    revert Errors.NegativePrice(token, address(feed[token]));
if (timestamp == 0)
    revert Errors...
if (answeredInRound < roundID)
    revert Errors...
```

5) ```getAllAccounts``` and ```accountsOwnedBy``` are only externally called functions, but you might consider adding pagination to them because the accounts array can only grow, it is never deleted.

