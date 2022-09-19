WATCHPUG
# Third-party investment rewards can not be claimed

## Summary

In the current implementation, claiming the rewards from the third-party protocols (Compound, Aave) is not supported.

## Vulnerability Detail

A significant portion of the investment gains from certain third-party protocols come from their incentive campaigns, therefore these rewards should be made claimable to the users.

Specifically:

1. There is no way to claim rewards from AaveV2/V3.
2. For Compound, the user can call `claimComp()` for the account to claim rewards. However, the reward tokens can not be added as an asset, this means that account's owner needs to either repay all the debt and withdraw them or swap them for a whitelisted asset.

## Impact

De facto fund loss as part of investment profits (in the form of rewards) is inaccessible.

## Code Snippet

`claimRewardsOnBehalf()` of AaveV2/V3 can only be called by whitelisted user (basically the depositor himself, i.e. account contract itself), which means that depositor can not receive rewards.

https://github.com/aave/aave-stake-v2/blob/22c56144efc1b0b0e7e7cc89f907f36fc3d53e0f/contracts/stake/AaveIncentivesController.sol#L129-L137

```solidity
  function claimRewards(
    address[] calldata assets,
    uint256 amount,
    address to
  ) external override returns (uint256) {
    require(to != address(0), 'INVALID_TO_ADDRESS');
    return _claimRewards(assets, amount, msg.sender, msg.sender, to);
  }

```

https://github.com/aave/aave-stake-v2/blob/22c56144efc1b0b0e7e7cc89f907f36fc3d53e0f/contracts/stake/AaveIncentivesController.sol#L139-L148

```solidity
  function claimRewardsOnBehalf(
    address[] calldata assets,
    uint256 amount,
    address user,
    address to
  ) external override onlyAuthorizedClaimers(msg.sender, user) returns (uint256) {
    require(user != address(0), 'INVALID_USER_ADDRESS');
    require(to != address(0), 'INVALID_TO_ADDRESS');
    return _claimRewards(assets, amount, msg.sender, user, to);
  }
```

https://github.com/aave/aave-v3-periphery/blob/13d063415d78035cc2d6f234c174f60442d3ca72/contracts/rewards/RewardsController.sol#L121-L143

```solidity
  /// @inheritdoc IRewardsController
  function claimRewards(
    address[] calldata assets,
    uint256 amount,
    address to,
    address reward
  ) external override returns (uint256) {
    require(to != address(0), 'INVALID_TO_ADDRESS');
    return _claimRewards(assets, amount, msg.sender, msg.sender, to, reward);
  }

  /// @inheritdoc IRewardsController
  function claimRewardsOnBehalf(
    address[] calldata assets,
    uint256 amount,
    address user,
    address to,
    address reward
  ) external override onlyAuthorizedClaimers(msg.sender, user) returns (uint256) {
    require(user != address(0), 'INVALID_USER_ADDRESS');
    require(to != address(0), 'INVALID_TO_ADDRESS');
    return _claimRewards(assets, amount, msg.sender, user, to, reward);
  }
```

In Compound's `Comptroller`:

Expert user can call `claimComp()` to claim COMP for him account.

Then they can use `UniV2Controller#swapErc20ForErc20()` to swap reward tokens to a whitelisted asset.

https://github.com/compound-finance/compound-protocol/blob/master/contracts/Comptroller.sol#L1345-L1366

```solidity
function claimComp(address[] memory holders, CToken[] memory cTokens, bool borrowers, bool suppliers) public {
    for (uint i = 0; i < cTokens.length; i++) {
        CToken cToken = cTokens[i];
        require(markets[address(cToken)].isListed, "market must be listed");
        if (borrowers == true) {
            Exp memory borrowIndex = Exp({mantissa: cToken.borrowIndex()});
            updateCompBorrowIndex(address(cToken), borrowIndex);
            for (uint j = 0; j < holders.length; j++) {
                distributeBorrowerComp(address(cToken), holders[j], borrowIndex);
            }
        }
        if (suppliers == true) {
            updateCompSupplyIndex(address(cToken));
            for (uint j = 0; j < holders.length; j++) {
                distributeSupplierComp(address(cToken), holders[j]);
            }
        }
    }
    for (uint j = 0; j < holders.length; j++) {
        compAccrued[holders[j]] = grantCompInternal(holders[j], compAccrued[holders[j]]);
    }
}
```

## Tool used

Manual Review

## Recommendation

Consider supporting `claimRewards()` / `claimComp()` in the controllers.