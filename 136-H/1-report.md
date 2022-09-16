0x52
# ERC4626Oracle.sol returns incorrect price if ERC4626.decimals != ERC4626.asset.decimals

## Summary

ERC4626Oracle.sol returns incorrect price if ERC4626.decimals != ERC4626.asset.decimals. An oracle return the incorrect value of an asset is always bad news and can lead to loss of funds for either users or the protocol

## Vulnerability Detail

[Oracle.sol#L35-L43](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/erc4626/ERC4626Oracle.sol#L35-L43)

    function getPrice(address token) external view returns (uint) {
        uint decimals = IERC4626(token).decimals();
        return IERC4626(token).previewRedeem(
            10 ** decimals
        ).mulDivDown(
            oracleFacade.getPrice(IERC4626(token).asset()),
            10 ** decimals
        );
    }

ERC4626 returns the amount of underlying asset received from redeeming, which will be returned in token.asset.decimals. The issue is that in L41 it is divided by token.decimals not token.asset.decimals. This means that when token.decimals != token.asset.decimals the answer will be returned with the wrong number of decimals. This is particularly bad considering a large number of ERC4626 vaults use USDC as their underlying which only has 6 decimals. 

For tokens in which token.decimals > token.asset.decimals, the tokens will be undervalued which could potentially cause unfair liquidation for the user. For tokens in which token.decimals < token.asset.decimals the tokens would be highly overvalued leading to gross over-borrowing and complete loss of all lender and protocol funds

## Impact

Potential loss of funds for both users and the protocol

## Code Snippet

## Tool used

Manual Review

## Recommendation

Normalize the output by the decimals of the underlying asset rather than the token as shown below:

    function getPrice(address token) external view returns (uint) {
        uint decimals = IERC4626(token).decimals();
        return IERC4626(token).previewRedeem(
            10 ** decimals
        ).mulDivDown(
            oracleFacade.getPrice(IERC4626(token).asset()),
            10 ** IERC4626(token).asset.decimals()
        );
    }
