xiaoming90
# Interest Accrued Can Be Stolen

## Summary

Interest accrued by the Liquidity Provider (LP) of a LToken vault can be stolen by a malicious user.

## Vulnerability Detail

Assume that at T0 (Time 0), Alice deposits 100 DAI into the DAI LToken vault and gets back 100 LP tokens

- Underlying balance of vault = 100 DAI

- Alice has 100 LP tokens


Assume that a T1, Charles decided to borrow all the `100` DAI from the vault. The state of the DAI LToken vault is as follows:

- `totalBorrowShares` (also known as `supply`)(total borrow shares minted) = 100 shares

- `borrows` (total amount of borrows) = 100 DAI

- Underlying balance of vault = 0 DAI

- `borrowsOf[Charles's account address]` = 100 shares

- Alice has 100 LP tokens

For simplicity's sake, assume that the borrowing rate is fixed at 1% per second and protocol reserve is ignored. The borrowing rate is intentionally magnified here to illustrate the issue.

Assume that one hour has passed without anyone triggering the vault's `updateState()` function. The `interestAccrued` would be `360` DAI if someone triggered the `updateState()` function.

Alice should be entitled to all the `360` DAI interest accrued because she is the only one who has put her asset into the vault to earn interest for the past hour. Therefore, if Alice decided to cash out now, she should get back her initial investment (100 DAI) + interest accrued (360 DAI), which is a total of `460` DAI. 

```solidity
getRateFactor() = 60 minutes * 60 seconds * 0.01 (1%) = 3.6

interestAccrued = borrows.mulWadUp(rateFactor);
interestAccrued = 100 * 3.6 = 360 DAI

borrows += interestAccrued;
borrows = 100 DAI + 360 DAI = 460 DAI
```
https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/tokens/LToken.sol#L217

```solidity
function getRateFactor() internal view returns (uint) {
    return (block.timestamp == lastUpdated) ?
        0 :
        ((block.timestamp - lastUpdated)*1e18)
        .mulWadUp(
            rateModel.getBorrowRatePerSecond(
                asset.balanceOf(address(this)),
                borrows
            )
        );
}
```

Assume that Alice decided not to cash out and no one has triggered the `updateState()` function or called any function that triggered the `updateState()` function yet. Thus, the state of DAI LToken vault remains unchanged.

Bob the attacker sees this opportunity and knows that the next state update will result in a huge increase in the vault's totalAsset because the `interestAccrued` (360 DAI) will be added to the `borrows` after the state update. Therefore, he decides to exploit this issue to extract most of the interest accrued.

The next few steps happen within the same transaction/block.

Bob flash-loan 1,000,000 DAI from dydx. He then deposits all of them into the DAI LToken vault. Based on the `convertToShares` formula, Bob will receive around `1000000` LP tokens

Note: Flash-loan fee in dydx is 2 Wei only which is considered insignificant for this example and is ignored.

```solidity
totalAsset() = asset.balanceOf(address(this)) + getBorrows() - getReserves()
totalAsset() = 0 + 100 DAI + 0 = 100 DAI

assets.mulDivDown(supply, totalAssets())
1000000.mulDivDown(100, 100) = 1000000 shares
```

Bob then called the `updateState` function to update the state of the vault. The `updateState` function is public and can be called by anyone.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/7ee416c9c0e312fdaa1605bd981775279fbc70b2/protocol/src/tokens/LToken.sol#L200

```solidity
function updateState() public {
    if (lastUpdated == block.timestamp) return;
    uint rateFactor = getRateFactor();
    uint interestAccrued = borrows.mulWadUp(rateFactor);
    borrows += interestAccrued;
    reserves += interestAccrued.mulWadUp(reserveFactor);
    lastUpdated = block.timestamp;
}
```

The state of the DAI LToken vault is as follows after the update:

- `totalBorrowShares` (also known as `supply`)(total borrow shares minted) = 100 shares

- `borrows` (total amount of borrows) = 460 DAI

- Underlying balance of vault = 1,00,000 DAI

- `borrowsOf[Charles's account address]` = 100 shares

- Alice has 100 LP tokens, and Bob has 1,000,000 LP tokens

Next, Bob proceeds to redeem all his 1,000,000 LP tokens. Based on the formula of `convertToAssets`, he will get back `1,000,359` DAI.

```solidity
totalAsset() = asset.balanceOf(address(this)) + getBorrows() - getReserves()
totalAsset() = 1000000 DAI + 460 DAI + 0 = 1000460 DAI

shares.mulDivDown(totalAssets(), supply);
1000000.mulDivDown(1000460 DAI, 1000100) = 1,000,359
```

Bob earns `359` DAI of the interest accrued from this exploit within a single block/transaction. Consequently, if Alice decided to cash out after Bob, she would only be entitled to `1` DAI of the interest accrued.

Note that Bob could also back-run and front-run any transaction that attempts to update the vault's state to achieve the same result.

## Impact

- Interest accrued that belongs to liquidity providers is stolen
- Unfair distribution of interest accrued

## Recommendation

Consider implementing the following measures to mitigate the issue:

- Use snapshotting to keep track of the deposit to ensure that interest accrued distributions are weighted according to deposit duration.
- Impose a withdrawal fee to make this attack unprofitable. In the past, hacks have happened due to a lack of withdrawal fee or once the withdrawal fee has been temporarily disabled. Refer to [here](https://github.com/yearn/yearn-security/blob/master/disclosures/2021-02-04.md#contributing-factors).
