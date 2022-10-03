bytehat
# USDT Fee-on-transfer will make protocol lose money

## Summary
Although USDT currently does not have a fee enabled on transfer, the USDT smart contract reserves the right to enable one in the future: https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code#L126. The protocol would lose money/break in various ways if this were to actually happen.

## Vulnerability Detail
For example, in [ERC4626.sol](https://github.com/sherlock-audit/2022-08-sentiment-bytehat-dev/blob/39f5a01e11b3570d4ce9a531c250db70df2ceb5f/protocol/src/tokens/utils/ERC4626.sol#L48), the amount of shares minted to the user depends only on the uint256 assets amount, which is possibly larger than the amount actually transferred in due to the fee. If the fee switch was enabled _after_ Sentiment goes live, then users that pay the fee will get the same number of shares as users that did not have to pay a fee, which is not fair.

## Impact
Many parts of the protocol will silently break if USDT (or any other token) enabled a fee-on-transfer.

## Code Snippet
```
    function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
        beforeDeposit(assets, shares);

        // Check for rounding error since we round down in previewDeposit.
        require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

        // Need to transfer before minting or ERC777s could reenter.
        asset.safeTransferFrom(msg.sender, address(this), assets);

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
    }
```

## Tool used
Manual Review

## Recommendation
Do balance checks before and after transfers to obtain the actual amount of tokens that are being transferred around in the protocol.
