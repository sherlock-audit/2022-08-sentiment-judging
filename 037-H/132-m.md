xiaoming90
# Yearn Oracle Is Vulnerable To Price Manipulation

## Summary

Yearn oracle is vulnerable to price manipulation. This allows an attacker to increase or decrease the price of Yearn's LP token to carry out various attacks against the protocol.

## Vulnerability Detail

The price oracle for Yearn's LP token is vulnerable to price manipulation. The value of the Yearn's LP token calculates based on the Yearn vault's `pricePerShare` which can be manipulated.

https://github.com/sherlock-audit/2022-08-sentiment-xiaoming9090/blob/99afd59bb84307486914783be4477e5e416510e9/oracle/src/yearn/YTokenOracle.sol#L38

```solidity
function getPrice(address token) external view returns (uint price) {
    address underlying_token = IYVault(token).token();
    price = IYVault(token).pricePerShare() *
        oracle.getPrice(underlying_token) /
        10 ** IYVault(underlying_token).decimals();
}
```

The same issue has been seen in the well-known hack of CREAM Finance in the past that resulted in the loss of 130m+ of assets. The root cause of the hack is that the price oracle of CREAM Finance relies on the Yearn vault's `pricePerShare` to determine the value of Yearn vault shares as collateral. This is exactly the same as what Sentiment's price oracle is doing. Therefore, under the right condition, a similar attack could be performed against Sentiment.

Following are some of the resources that provided a detailed overview of the CREAM Finance hack and root cause analysis:

- https://mudit.blog/cream-hack-analysis/
- https://github.com/yearn/yearn-security/blob/master/disclosures/2021-10-27.md
- https://medium.com/cream-finance/post-mortem-exploit-oct-27-507b12bb6f8e

This is a sophisticated attack that involves multiple calculations and requires careful planning to achieve the desired outcome. The outcome is also dependent on various market and economic factors. Thus I will only provide a simplified high-level overview of the attack to illustrate the issue:

Assume that the XYZ yVault with the following configuration:

- Total Asset = 100 XYZ
- Total Supply = 100 shares
- pricePerShare = 1
- LP Token = yXYZ

Note: In the CREAM hack, the attacker managed to reduce the `pricePerShare ` to 1 and reduce the total supply by obtaining a large amount of yUSD LP tokens and redeeming/burning them. The smaller the total supply, the cheaper it will be for the attacker to manipulate the `pricePerShare` as it is calculated based on the `Total Asset/Total Supply` formula.

The attacker flash-loan some DAI and use them to obtain 1m yXYZ LP tokens from the open market. He proceeds to deposit all 1m yXYZ LP tokens into Sentiment as collateral. At this point, the attacker has 1m worth of collateral in his Sentiment account.

However, the attacker can donate/airdrop 100XYZ to the XYZ yVault immediately increasing the `pricePerShare` twofold, and the state of XYZ yVault will be as follows after the donation/airdrop:

- Total Asset = 200 XYZ
- Total Supply = 100 shares
- pricePerShare = 200 XYZ/100 shares = 2
- LP Token = yXYZ

Sentiment uses `pricePerShare` to determine the value of Yearn vault shares as collateral. As the value of its collateral now has effectively doubled as far as the Sentiment can tell, the attacker's Sentiment account now has twice as much borrowing power, allowing the attacker to borrow more assets from Sentiment compared to before the donation/airdrop.

## Impact

The attacker could perform price manipulation to make the apparent value of an asset to be much higher or much lower than the true value of the asset. Following are some risks of price manipulation:

- An attacker can increase the value of their collaterals to increase their borrowing power so that they can borrow more assets than they are allowed from Sentiment.
- An attacker can decrease the value of some collaterals and attempt to liquidate another user account prematurely.

## Recommendation

The key vulnerability lies in the price calculation of a yield-bearing token that can have its value increased in the same transaction by donating to the vault to increase its `pricePerShare`. Therefore, it is recommended to:

- Avoid using the `pricePerShare` to calculate the price of Yearn's LP token
- Avoid accepting or supporting any such token as collateral to mitigate the risk of a price manipulation attack. Learning from the CREAM hack that I mentioned earlier, they have decided not to support such tokens moving forward. Refer to [here](https://creamdotfinance.medium.com/moving-forward-post-exploit-next-steps-for-c-r-e-a-m-finance-1ad05e2066d5)