__141345__
# UniSwap V3 TWAP oracle delay could cause fund loss

## Summary

UniSwap V3 TWAP price feed will have some delay relative to spot price, according to the TWAP definition, it is an average over some time window. In severe market conditions, when some asset plunges, the price delay may incur loss to the protocol.



## Vulnerability Detail

When the market crashes, it is possible that the price drop over 10% in a couple of minutes, especially for the tokens with smaller market cap. Even for some big market cap tokens, the crash can be a free fall, like UST/luna. 

Assuming token A price crashes, due to the delay, the difference between the UniSwap V3 TWAP oracle price feed and spot price is 20%. Some user put token A as collateral and borrow to the limit. The actual `balance/borrow` ratio is 1, much lower than the 1.2 threshold. 

The problem is liquidation won't be profitable. As a results, liquidation mechanism will stop working to keep the protocol solvent. Liquidators repay the loan but receive less than what they paid for. As no liquidation is performed, the loss might keep on growing. Even the delay is less than 20%, during market crash, liquidators are not likely to risk a big loss when the anticipation being pessimistic, since the collateral might drop to zero, like UST/luna case. If liquidation cannot function timely but just stop working, the loss will be out of control.



## Impact

The protocol could loss fund in market crash due to the TWAP delay. The liquidation mechanism might not efficiently help the protocol keep solvent. After the crash grow into certain degree, the protocol loss will be out of control. 


## Code Snippet

The price feed from the UniSwap V3 TWAP oracle price would have some delay by definition:
```solidity
// oracle\src\uniswap\UniV3TWAPOracle.sol
    function getPrice(address token) public view returns (uint256) {
        // ...
        (int24 arithmeticMeanTick, ) = OracleLibrary.consult(
            pool,
            twapPeriod
        );
        // ...
    }
```


## Tool used

Manual Review

## Recommendation

- provide additional incentive for liquidation, in case of market crash
- buy insurance as downside protection, for the assets using the UniSwap V3 TWAP oracle
