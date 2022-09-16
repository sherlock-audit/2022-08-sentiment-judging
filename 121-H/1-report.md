xiaoming90
# Missing State Update Causing More Shares To Be Minted And Fewer Assets To Be Returned

## Summary

The `depsoitEth` and `redeemEth` functions do not update the state of the vault before minting and burning, thus causing more shares to be minted and fewer assets to be returned during redemption.

## Vulnerability Detail

#### Instance 1 - LEther's deposit

Within the LEther vault, there are two methods that a user can deposit their assets:

1) Call the `LEther.depositEth` function and attach Ethers to the transaction. The `LEther.depositEth` function will automatically wrap the Ether into WETH and mint the shares for the user.
2) Since `LEther` contract inherits the `ERC4626` contract, it also inherits the `deposit` function. A user can call `LEther.deposit` function and transfer WETH to the LEther vault and the contract will mint the shares for the user.

When the `LEther.deposit` function is called, the `beforeDeposit` function at Line 49 of the `ERC4626` contract will be triggered to update the state of the vault to ensure that the `borrows` and `reserves` state variables are up-to-date and the interest accrued has been accounted for. 

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

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L239

```solidity
File: LToken.sol
239:     function beforeDeposit(uint, uint) internal override { updateState(); }
```

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L200

```solidity
File: LToken.sol
199:     /// @notice Updates state of the lending pool
200:     function updateState() public {
201:         if (lastUpdated == block.timestamp) return;
202:         uint rateFactor = getRateFactor();
203:         uint interestAccrued = borrows.mulWadUp(rateFactor);
204:         borrows += interestAccrued;
205:         reserves += interestAccrued.mulWadUp(reserveFactor);
206:         lastUpdated = block.timestamp;
207:     }
```

However, the `LEther.depositEth` function does not update the state of the vault before minting the shares.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LEther.sol#L31

```solidity
File: LEther.sol
31:     function depositEth() external payable {
32:         uint assets = msg.value;
33:         uint shares = previewDeposit(assets);
34:         require(shares != 0, "ZERO_SHARES");
35:         IWETH(address(asset)).deposit{value: assets}();
36:         _mint(msg.sender, shares);
37:         emit Deposit(msg.sender, msg.sender, assets, shares);
38:     }
```

#### Instance 2 - LEther's Redeem

Within the LEther vault, there are two methods that a user can redeem their assets:

1) Call the `LEther.redeemEth` function. The `LEther.redeemEth` function burns the shares and unwraps the WETH to Ether and sends it to the user.
2) Since `LEther` contract inherits the `ERC4626` contract, it also inherits the `redeem` function. A user can call `LEther.redeem` function, and it will burn the shares and transfer the WETH to the user.

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

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LToken.sol#L240

```solidity
File: LToken.sol
240:     function beforeWithdraw(uint, uint) internal override { updateState(); }
```

However, the `LEther.redeemEth()` function does not update the state of the vault before minting the shares.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/protocol/src/tokens/LEther.sol#L47

```solidity
File: LEther.sol
47:     function redeemEth(uint shares) external {
48:         uint assets = previewRedeem(shares);
49:         _burn(msg.sender, shares);
50:         emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
51:         IWETH(address(asset)).withdraw(assets);
52:         msg.sender.safeTransferEth(assets);
53:     }
```

## Impact

Note that when the `updateState` function is executed, the value in `borrows` and `reserves` state variables will increase as the interest accrued is added. 

Therefore, if the `updateState` is not executed, the `borrows` and `reserves` state variables will be undervalued,  Consequently, it will cause the `totalAssets` of the vault will be undervalued because `totalAssets` is calculated based on the total underlying asset, total borrow, and reserve of the vault.

The impact of this issue related to depositing is as follows:

- Since `totalAssets` is undervalued, the `previewDeposit` will return a larger value, thus causing more shares to be minted. As a result, the user will end up getting more shares.
- Inconsistent in the issuance of shares because users who called the `despoitEth` function are given more shares compared to users who called the `deposit` function in the LEther vault.

- This breaks the internal accounting of the LEther vault, and it is unfair to exist Liquidity providers of the affected vault whose shares get diluted due to accounting errors.

The impact of this issue related to redemption is as follows:

- Since `totalAssets` is undervalued, the `previewRedeem` will return a smaller value, thus causing fewer assets to be returned to the user. As a result, the user will end up getting few assets when they burn/redeem their shares.
- Inconsistent in the distribution of assets because users who called the `redeemEth` function will receive more assets compared to users who called the `redeem` function in the LEther vault.

- This breaks the internal accounting of the LEther vault, and it is unfair to exist Liquidity providers of the affected vault as their assets get distributed inappropriately.

If the state of a vault has not been updated for a long time, the impact of this issue will be serious. In this case, the `totalAssets` will be significantly undervalued when performing the calculation.

## Recommendation

Within the `depsoitEth` and `redeemEth` functions, update the state of the vault before minting or burning the shares.

```diff
function depositEth() external payable {
+	updateState();
    uint assets = msg.value;
    uint shares = previewDeposit(assets);
    require(shares != 0, "ZERO_SHARES");
    IWETH(address(asset)).deposit{value: assets}();
    _mint(msg.sender, shares);
    emit Deposit(msg.sender, msg.sender, assets, shares);
}

function redeemEth(uint shares) external {
+	updateState();
    uint assets = previewRedeem(shares);
    _burn(msg.sender, shares);
    emit Withdraw(msg.sender, msg.sender, msg.sender, assets, shares);
    IWETH(address(asset)).withdraw(assets);
    msg.sender.safeTransferEth(assets);
}
```