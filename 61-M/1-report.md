cergyk
# LToken vault is Not Compatible with Fee Tokens

## [M] LToken vault is Not Compatible with Fee Tokens
### Problem
Some ERC20 tokens charge a transaction fee for every transfer (used to encourage staking, add to liquidity pool, pay a fee to contract owner, etc.). If any such token is used when depositing or repaying a debt, the LToken vault will always receive less and contract would lose economic value.

### Proof of Concept
Plenty of ERC20 tokens charge a fee for every transfer (e.g. Safemoon and its forks), in which the amount of token received is less than the amount being sent. When a fee token is used as the `asset` in the `LToken` contract, the amount received by the contract would be less than the amount being sent. To be more precise, functions
```solidity
protocol/src/tokens/utils/ERC4626.sol
48:     function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {

62:     function mint(uint256 shares, address receiver) public virtual returns (uint256 assets) {

protocol/src/tokens/LToken.sol
153:     function collectFrom(address account, uint amt)
```
do not check how amount of tokens received and as result contract will lose economic value.
###  Mitigation 
Check amount of tokens received or disallow fee tokens from being used in the vault. 