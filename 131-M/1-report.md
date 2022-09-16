xiaoming90
# Internal Accounting Issue Due To Fee-On-Transfer/Rebasing Tokens 

## Summary

Internal accounting issues will occur when the fee-on-transfer or rebasing token is supported in the future.

## Vulnerability Detail

> Note: Per the Sentiment team's clarification in the Discord channel, it was understood that as of 09 September, the team has no plans to integrate with any fee-on-transfer/rebasing tokens as a Day 1 feature and has not made any special provisions around the same. However, it was understood that they would definitely want to allow these types of tokens in the future. Thus, it is recommended to future-proof the contracts so that such tokens can be supported later.

LToken vault that supports Fee-On-Transfer/Rebasing tokens will result in internal accounting issues during depositing and minting.

Assume that `XYZ` token is a fee-on-transfer token with a 10% transfer fee.

#### Instance #1 - Deposit

Assume that the user deposits 100 XYZ tokens. The number of shares minted will be based on 100 XYZ tokens, but the actual amount of XYZ tokens received by the vault will only be 90 XYZ tokens.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/utils/ERC4626.sol#L48

```solidity
File: ERC4626.sol
48:     function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
49:         beforeDeposit(assets, shares);
50: 
51:         // Check for rounding error since we round down in previewDeposit.
52:         require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");
53: 
54:         // Need to transfer before minting or ERC777s could reenter.
55:         asset.safeTransferFrom(msg.sender, address(this), assets);
56: 
57:         _mint(receiver, shares);
58: 
59:         emit Deposit(msg.sender, receiver, assets, shares);
60:     }
```

#### Instance #2 - Minting

Assuming that the user wants to mint 10 shares. The `previewMint` function determines that 100 XYZ tokens are needed to mint 10 shares. The vault proceeds to mint 10 shares to the user, but the actual amount of XYZ tokens received by the vault will only be 90 XYZ tokens.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/utils/ERC4626.sol#L62

```solidity
File: ERC4626.sol
62:     function mint(uint256 shares, address receiver) public virtual returns (uint256 assets) {
63:         beforeDeposit(assets, shares);
64: 
65:         assets = previewMint(shares); // No need to check for rounding error, previewMint rounds up.
66: 
67:         // Need to transfer before minting or ERC777s could reenter.
68:         asset.safeTransferFrom(msg.sender, address(this), assets);
69: 
70:         _mint(receiver, shares);
71: 
72:         emit Deposit(msg.sender, receiver, assets, shares);
73:     }
```

## Impact

Internal accounting issues during depositing and minting. The vault mints the shares based on the number of incoming assets input by the users (e.g. 100 XYZ in the above examples), but in reality, it receives fewer tokens than expected (e.g. 90 XYZ in the above examples).

## Recommendation

Consider comparing the before and after `balanceOf` to get the actual transferred amount, and calculate the number of shares to be minted based on the actual transferred amount.