carrot
# Non Liquidatable Accounts

## Summary
Accounts can be made non-liquidatable by using malicious tokens as assets
## Vulnerability Detail
Account can be made to interact with malicious LP token, reverting liquidate function calls. This can lead to bad debt, and loss of assets from liquidity providers since these assets cannot be recovered.
## Impact
High
## Code Snippet
Consider the following case: 
1. Bad actor creates LP pair with a token with a blacklist (token cannot be transferred out if msg.sender == address(account)).
2. Actor then sends the LP pair to the account
3. Actor calls "remove liquidity" function with this LP token, which doesn't check incoming or outgoing tokens, adding blacklisted tokens to **assets** list
4. Actor cannot be liquidated, as sweepTo function will fail due to the blacklisted coins in assets list

Detailed Explanation:
controller/src/uniswap/UniV2Controller.sol:
```solidity
function removeLiquidity(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address tokenA, address tokenB) = abi.decode(data, (address, address));

        address[] memory tokensIn = new address[](2);
        tokensIn[0] = tokenA;
        tokensIn[1] = tokenB;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);

        return(true, tokensIn, tokensOut);
    }
```
This function is responsible for checking if the removeLiquidity function is allowed to be called by the account. However, this function does not check if the tokens coming into the account are allowed tokens or not. A bad actor can create an LP token where the underlying tokens have a modified erc20 contract which reverts if a particular address tries to move coins out of the account (very common in rugpull/ honeypot tokens). By calling removeLiquidity, we can get such a token into the account **and added to the assets list**.

After this function call, the next code snippet is run
protocol/src/core/AccountManager.sol:
```solidity
function exec(
        address account,
        address target,
        uint amt,
        bytes calldata data
    )
        external
        onlyOwner(account)
    {
        bool isAllowed;
        address[] memory tokensIn;
        address[] memory tokensOut;
        (isAllowed, tokensIn, tokensOut) =
            controller.canCall(target, (amt > 0), data);
        if (!isAllowed) revert Errors.FunctionCallRestricted();
        _updateTokensIn(account, tokensIn);
        (bool success,) = IAccount(account).exec(target, amt, data);
        if (!success)
            revert Errors.AccountInteractionFailure(account, target, amt, data);
        _updateTokensOut(account, tokensOut);
        if (!riskEngine.isAccountHealthy(account))
            revert Errors.RiskThresholdBreached();
    }
```
Right after calling the canCall function, we have some malicious tokens in the variable tokensIn. These tokens are then added as an asset without checking if they are allowed in _updateTokensIn function
protocol/src/core/AccountManager.sol:
```solidity
function _updateTokensIn(address account, address[] memory tokensIn)
        internal
    {
        uint tokensInLen = tokensIn.length;
        for(uint i; i < tokensInLen; ++i) {
            if (IAccount(account).hasAsset(tokensIn[i]) == false)
                IAccount(account).addAsset(tokensIn[i]);
        }
    }
``` 
We have established:
1. Tokens exist with blacklists making it impossible to move them out of blacklisted accounts
2. Those tokens have been added into the assets list

When calling liquidate, the final step is sweepTo function which will fail due to these blacklisted assets
protocol/src/core/Account.sol:
```solidity
function sweepTo(address toAddress) external accountManagerOnly {
        uint assetsLen = assets.length;
        for(uint i; i < assetsLen; ++i) {
            assets[i].safeTransfer(
                toAddress,
                assets[i].balanceOf(address(this))
            );
            hasAsset[assets[i]] = false;
        }
        delete assets;
        toAddress.safeTransferEth(address(this).balance);
    }
```
This function will always revert if blacklisted tokens are added as an asset, making liquidation impossible, and funds upto 5 times the collateral value will be locked into this account forever.

## Tool used

Manual Review
Foundry (POC in development)

## Recommendation
Possible solutions:
1. Check for allowed tokens when removing liquidity (for both **removeLiquidity** and **removeLiquidityEth**)

```solidity
function removeLiquidity(bytes calldata data)
        internal
        view
        returns (bool, address[] memory, address[] memory)
    {
        (address tokenA, address tokenB) = abi.decode(data, (address, address));

        address[] memory tokensIn = new address[](2);
        tokensIn[0] = tokenA;
        tokensIn[1] = tokenB;

        address[] memory tokensOut = new address[](1);
        tokensOut[0] = UNIV2_FACTORY.getPair(tokenA, tokenB);

        bool check = controller.isTokenAllowed(tokensIn[0]) && controller.isTokenAllowed(tokensIn[1]);
        return(check, tokensIn, tokensOut);
    }
```

2. Check against pre-specified list before adding as an asset in _updateTokensIn function. This list should not be the same as isCollateralAllowed, since this list should also include aave tokens, legit LP tokens, etc.


