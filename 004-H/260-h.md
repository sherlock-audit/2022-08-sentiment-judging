IllIllI
# First depositor can break minting of LPToken shares

## Summary
The first depositor can get shares, then deposit a large 'donation' such that later `balanceOf()` calculations show that each share is worth a lot more than the depositor originally paid.

## Vulnerability Detail
1 Depositor A deposits 2 wei, gets 2 shares
2 Depositor A makes a 'donation' of $200,000 worth of the token by calling `token.transfer()` to the LPToken's address. 
3 Depositor B deposits $50 worth of the token, but gets zero shares, since according to `balanceOf()`, one share costs $200k, and $50 rounds down to zero shares

## Impact
Later depositors deposit funds and get zero LPTokens. The original depositor's shares grow in value instead

## Code Snippet
deposit mints shares based on preview:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/tokens/utils/ERC4626.sol#L48-L60

preview is based on converting to shares:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/tokens/utils/ERC4626.sol#L138-L140

shares is based on total assets:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/tokens/utils/ERC4626.sol#L126-L130

total assets for LPTokens use the balance of the contract:
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/tokens/LToken.sol#L191-L193

## Tool used

Manual Review

## Recommendation
Uniswap V2 solved this problem by sending the first 1000 LP tokens to the zero address
https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L124