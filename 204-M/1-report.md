GalloDaSballo
# M-03 YTokenOracle doesn't account for losses when pricing the yToken

## Summary

Yearn vaults use multiple strategies, some of them can incurr a loss, the loss will be charged to the caller when withdrawing (the caller being the protocol).

The [`Ytoken.getPrice`](https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/yearn/YTokenOracle.sol#L38) incorrectly assumes that a withdrawal of 10 ** decimals is equivalent to a bigger withdrawal.

Because of the Yearn V2 codebase, we can see that a loss may happen on withdraw: https://github.com/yearn/yearn-vaults/blob/efb47d8a84fcb13ceebd3ceb11b126b323bcc05d/contracts/Vault.vy#L1074

Meaning that using the 10 ** decimals price for quoting is incorrect as it doesn't represent the total liquidatable assets, which may be very different (especially for tokens that use multiple strategies, especially levered strategies such as Levered AAVE, Levered COMP etc..)

## Vulnerability Detail

- Attacker knows vault can only withdraw up to 50% of the underlying before getting rekt
- Attacker borrows 100% of token value, the oracle allows this as the oracle only checks for the price of 10 ** decimals tokens
- Attacker purposefully get's liquidated
- The system withdraws from the yearn vault and incurs a loss
- Attacker has ran away with more value than intended
- Additionally, self-liquidation may be used as if the Yearn Vault is using any liquidity pool to liquidate to token, the imbalance caused by the withdrawal could itself create an opportunity for an arbitrage

## Impact

Oracle will allow to borrow more than what is liquidatable from the yVault


## Tool used

Manual Review

## Recommendation

Refactor `getPrice(address)` to `getPrice(address, amount)` and pass in the exact amount to ensure it can be liquidated fully without a loss

Alternatively cap the Yearn tokens to a massively small LTV (35% is probably as high as you should go without simming on a token by token basis)

## Sentiment Team
Not fixing since we don't plan to launch with Yearn. We are okay with not including this contract in the coverage scope since it won't be deployed.

## Lead Senior Watson
Acknowledged. 
