Chom
# Chainlink’s `latestRoundData` might return stale or incorrect results

## Summary
Chainlink’s `latestRoundData` might return stale or incorrect results

## Vulnerability Detail

We are using latestRoundData, but there is no check if the return value indicates stale data. These checks include `answeredInRound >= roundID`, `answer > 0` and `block.timestamp <= updatedAt + stalePriceDelay` but only `answer > 0` in check.

```solidity
 function getPrice(address token) external view virtual returns (uint) { 
     (, int answer,,,) = 
         feed[token].latestRoundData(); 
  
     if (answer < 0) 
         revert Errors.NegativePrice(token, address(feed[token])); 
  
     return ( 
         (uint(answer)*1e18)/getEthPrice() 
     ); 
 } 
```

This could lead to stale prices according to the Chainlink documentation:

https://docs.chain.link/docs/historical-price-data/#historical-rounds
https://docs.chain.link/docs/faq/#how-can-i-check-if-the-answer-to-a-round-is-being-carried-over-from-a-previous-round

## Impact
Stale or incorrect results have very severe effects. For example, LUNA has crashed down to $0.00001 but chainlink price still says that LUNA price is $0.01 leads to a big arbitrage opportunity by buying LUNA from market -> supply LUNA -> borrow stable coins out of the pool.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/oracle/src/chainlink/ChainlinkOracle.sol#L49-L59

https://github.com/sherlock-audit/2022-08-sentiment-Chomtana/blob/db4ce63352bcfd2bde50b43182f5a27f60563f5d/oracle/src/chainlink/ChainlinkOracle.sol#L65-L73

## Tool used

Manual Review

## Recommendation

Perform all necessary checks

```solidity
uint256 constant stalePriceDelay = 10 minutes;

 function getPrice(address token) external view virtual returns (uint) { 
     (uint80 roundID, int256 _price, , uint256 updatedAt, uint80 answeredInRound) = 
         feed[token].latestRoundData(); 
  
     if (answer < 0) 
         revert Errors.NegativePrice(token, address(feed[token])); 

     if (answeredInRound < roundID) revert Errors.StalePrice(token, address(feed[token]));

     if (block.timestamp > updatedAt + stalePriceDelay) revert Errors.StalePrice(token, address(feed[token]));
  
     return ( 
         (uint(answer)*1e18)/getEthPrice() 
     ); 
 } 
```

Apply this pattern to `getEthPrice` too


