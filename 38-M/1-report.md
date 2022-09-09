minera
# contract should check the responses from chainlink aggregator 

## Summary
On ChainlinkOracle.sol, contract use the latestRoundData, but there is no check if the return value indicates stale/(manipulated) data.

## Vulnerability Detail

Stale prices could put funds at risk. According to Chainlink's documentation, This function does not error if no answer has been reached but returns 0, causing an incorrect price fed to the PriceOracle. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to the index price calculations of the AMM. Oracle reliance has historically resulted in crippled on-chain systems, and complications that lead to these outcomes can arise from things as simple as network congestion.


## Impact
The getPrice and getEthprice function in the contract ChainlinkOracle.sol
fetches the asset price from a Chainlink aggregator using the latestRoundData function. However, there are no checks on roundID or timeStamp, resulting in stale prices. The oracle wrapper calls out to a chainlink oracle receiving the latestRoundData(). It then checks freshness by verifying that the answer is indeed for the last known round.

If there is a problem with chainlink starting a new round and finding consensus on the new value for the oracle (e.g. chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) consumers of this contract may continue using outdated stale data (if oracles are unable to submit no new round is started)
can cause loss of funds.

## Code Snippet

https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L49
https://github.com/sentimentxyz/oracle/blob/59b26a3d8c295208437aad36c470386c9729a4bc/src/chainlink/ChainlinkOracle.sol#L65

## Tool used

Manual Review

## Recommendation
add RoundID, and answeredInRound, timestamp for checks.

```
require(answeredInRound >= roundID, "...");
require(timeStamp != 0, "...");

```
And do your code like this 

```
function getPrice(address token) external view virtual returns (uint) {
        (uint80 roundID, int answer, , uint256 timeStamp, unit 80 answeredInRound) =
            feed[token].latestRoundData();


            /// white hat comment: this will checks if the answer to a round is being carried over from a previous round

      require(answeredInRound >= roundID, "ERR");

            /// white hat comment: checks if timestamp response is zero

      require(timeStamp != 0, "ERR");
    
    
      if (answer < 0)
            revert Errors.NegativePrice(token, address(feed[token]));
 
        return (
            (uint(answer)*1e18)/getEthPrice()
        );
    }





```
## References
Medium Severity Issue From The FEI Protocol:

https://consensys.net/diligence/audits/2021/09/fei-protocol-v2-phase-1/#chainlinkoraclewrapper-latestrounddata-might-return-stale-results
This could lead to stale prices according to the Chainlink documentation:

https://docs.chain.link/docs/historical-price-data/#historical-rounds

