xiaoming90
# State Update At The Wrong Sequence Causing Fewer Assets To Be Returned

## Summary

The `redeem` function does not perform a state update before executing the `previewRedeem`. Instead, the function perform a state update after executing the `previewRedeem` function. As a result, the `previewRedeem` function performs its calculation based on outdated information and cause the user will end up getting few assets when they burn/redeem their shares.

## Vulnerability Detail

When the `LEther.redeem` function is called, the `beforeWithdraw` function at Line 111 of the `ERC4626` contract will be triggered to update the state of the vault to ensure that the `borrows` and `reserves` state variables are up-to-date and the interest accrued has been accounted for. 

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/utils/ERC4626.sol#L97

```solidity
File: ERC4626.sol
097:     function redeem(
098:         uint256 shares,
099:         address receiver,
100:         address owner
101:     ) public virtual returns (uint256 assets) {
102:         if (msg.sender != owner) {
103:             uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.
104: 
105:             if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
106:         }
107: 
108:         // Check for rounding error since we round down in previewRedeem.
109:         require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");
110: 
111:         beforeWithdraw(assets, shares);
112: 
113:         _burn(owner, shares);
114: 
115:         emit Withdraw(msg.sender, receiver, owner, assets, shares);
116: 
117:         asset.safeTransfer(receiver, assets);
118:     }
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L239

```solidity
File: LToken.sol
239:     function beforeDeposit(uint, uint) internal override { updateState(); }
```

However, the sequence of the state update is wrong as the state update should be performed before the `previewRedeem` function is executed.

## Impact

Note that when the `updateState` function is executed, the value in `borrows` and `reserves` state variables will increase as the interest accrued is added. 

Therefore, if the `updateState` is not executed, the `borrows` and `reserves` state variables will be undervalued,  Consequently, it will cause the `totalAssets` of the vault will be undervalued because `totalAssets` is calculated based on the total underlying asset, total borrow, and reserve of the vault.

Since `totalAssets` is undervalued, when the `previewRedeem` function at Line 109 of `ERC4626` contract is executed, the `previewRedeem` will return a smaller value. Thus, this causes fewer assets to be returned to the user. As a result, the user will end up getting few assets when they burn/redeem their shares.

If the state of a vault has not been updated for a long time, the impact of this issue will be serious. In this case, the `totalAssets` will be significantly undervalued, causing users to lose a large number of assets during redemption.

## Recommendation

It is recommended to execute the `beforeWithdraw` function before calling the `previewRedeem` function so that the `previewRedeem` function will perform its calculation based on the latest vault state.

```diff
function redeem(
    uint256 shares,
    address receiver,
    address owner
) public virtual returns (uint256 assets) {
    if (msg.sender != owner) {
        uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.

        if (allowed != type(uint256).max) allowance[owner][msg.sender] = allowed - shares;
    }

+	beforeWithdraw(assets, shares);

    // Check for rounding error since we round down in previewRedeem.
    require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");

-   beforeWithdraw(assets, shares);

    _burn(owner, shares);

    emit Withdraw(msg.sender, receiver, owner, assets, shares);

    asset.safeTransfer(receiver, assets);
}
```