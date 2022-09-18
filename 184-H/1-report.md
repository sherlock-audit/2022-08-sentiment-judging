rbserver
# Chainlink oracle data feeds are not sufficiently validated and can lead to incorrect account actions

## Summary
Currently, because the Chainlink oracle data feeds are not sufficiently validated and can return stale answers, incorrect account actions can be done based on these stale answers.

## Vulnerability Detail
As shown in the Code Snippet section:
1. The `getPrice` and `getEthPrice` functions only check if `answer < 0`, where `answer` is returned by the Chainlink oracle data feed's `latestRoundData` function. However, other returned values are not used for ensuring that the returned answer is not stale.
2. The returned answer is used in the `_valueInWei` function, which is used in the `isBorrowAllowed`, `isWithdrawAllowed`, `_getBalance`, and `_getBorrows` functions.
3. `isBorrowAllowed`, `isWithdrawAllowed`, `_getBalance`, and `_getBorrows` are critical for determining if a user can borrow, withdraw, settle, and/or liquidate the relevant account. Hence, the returned answer that is stale can lead to incorrect account actions.

## Impact
Since the stale answers returned by the Chainlink oracle data feeds that are not sufficiently validated can lead to incorrect account actions, critical account actions that should not occur can be performed. For example, a user who should not be able to borrow anymore based on a returned answer that is not stale can be allowed to borrow more when the stale answer is used. For another instance, an account that should not be liquidable based on a returned answer that is not stale can be liquidated unexpectedly when the stale answer is used.

## Code Snippet
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ArbiChainlinkOracle.sol#L47-L59
```solidity
    function getPrice(address token) external view override returns (uint) {
        if (!isSequencerActive()) revert Errors.L2SequencerUnavailable();

        (, int answer,,,) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L59
```solidity
    function getPrice(address token) external view virtual returns (uint) {
        (, int answer,,,) =
            feed[token].latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L65-L73
```solidity
    function getEthPrice() internal view returns (uint) {
        (, int answer,,,) =
            ethUsdPriceFeed.latestRoundData();

        if (answer < 0)
            revert Errors.NegativePrice(address(0), address(ethUsdPriceFeed));

        return uint(answer);
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L178-L188
```solidity
    function _valueInWei(address token, uint amt)
        internal
        view
        returns (uint)
    {
        return oracle.getPrice(token)
        .mulDivDown(
            amt,
            10 ** ((token == address(0)) ? 18 : IERC20(token).decimals())
        );
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L72-L86
```solidity
    function isBorrowAllowed(
        address account,
        address token,
        uint amt
    )
        external
        view
        returns (bool)
    {
        uint borrowValue = _valueInWei(token, amt);
        return _isAccountHealthy(
            _getBalance(account) + borrowValue,
            _getBorrows(account) + borrowValue
        );
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L98-L112
```solidity
    function isWithdrawAllowed(
        address account,
        address token,
        uint amt
    )
        external
        view
        returns (bool)
    {
        if (IAccount(account).hasNoDebt()) return true;
        return _isAccountHealthy(
            _getBalance(account) - _valueInWei(token, amt),
            _getBorrows(account)
        );
    }
```

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/RiskEngine.sol#L150-L176
```solidity
    function _getBalance(address account) internal view returns (uint) {
        address[] memory assets = IAccount(account).getAssets();
        uint assetsLen = assets.length;
        uint totalBalance;
        for(uint i; i < assetsLen; ++i) {
            totalBalance += _valueInWei(
                assets[i],
                IERC20(assets[i]).balanceOf(account)
            );
        }
        return totalBalance + account.balance;
    }

    function _getBorrows(address account) internal view returns (uint) {
        if (IAccount(account).hasNoDebt()) return 0;
        address[] memory borrows = IAccount(account).getBorrows();
        uint borrowsLen = borrows.length;
        uint totalBorrows;
        for(uint i; i < borrowsLen; ++i) {
            address LTokenAddr = registry.LTokenFor(borrows[i]);
            totalBorrows += _valueInWei(
                borrows[i],
                ILToken(LTokenAddr).getBorrowBalance(account)
            );
        }
        return totalBorrows;
    }
```

## Tool used
Manual Review

## Recommendation
According to the [Chainlink's documentation](https://docs.chain.link/docs/historical-price-data/#getrounddata-return-values):
1. `roundId` and `answeredInRound` are also returned. "You can check `answeredInRound` against the current `roundId`. If `answeredInRound` is less than `roundId`, the answer is being carried over. If `answeredInRound` is equal to `roundId`, then the answer is fresh."
2. "A read can revert if the caller is requesting the details of a round that was invalid or has not yet been answered. If you are deriving a round ID without having observed it before, the round might not be complete. To check the round, validate that the timestamp on that round is not 0."

Therefore, for example, https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49-L59 can be updated to the following code.
```solidity
    function getPrice(address token) external view virtual returns (uint) {
        (uint80 roundId, int256 answer, , uint256 updatedAt, uint80 answeredInRound) =
            feed[token].latestRoundData();

        require(answeredInRound >= roundId, "answer is stale");
        require(updatedAt > 0, "round is incomplete");

        if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));

        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }
```