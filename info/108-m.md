0xNazgul
# [NAZ-M1] Two Transaction Initialization of Smart Contracts Can be Front-run

## Summary
The deployment process of upgradeable contracts can be front-run by malicious users.

## Vulnerability Detail
In the [arbiDeploymentFlow.md](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/deployments/ArbiDeploymentFlow.md) it is stated that the deployment flow will consist of Two transaction initialization. Anyone can front-run initialization and feed any inputs. 
* Loss of funds.
* Failure of the protocol, with the need for redeploy.
* Loss of control over protocol elements (some smart contracts).
* The possibility of replacing contracts and settings with harmful ones.
* And other things that come out of it...

## Impact

Example for `LToken.sol`.
1. Mallory listens for deployment transaction in mempool, etherscan, transaction in block etc.
2. Sends the transaction for `init()` her own parameters in which they go unnoticed.
3. Alice interacts with Mallory's `LToken.sol` and lose funds.

## Code Snippet

[`Account.sol#L59`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/Account.sol#L59), [`AccountManager.sol#L68`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/AccountManager.sol#L68), [`Registry.sol#L61`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/Registry.sol#L61), [`LToken.sol#L88`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/tokens/LToken.sol#L88)

## Tool used

Manual Review

## Recommendation
Carry out checks at the initialization stage or redesign the deployment process with the initialization of contracts during deployment. Some options:

1. Make deployments+init in one transaction via multisig wallet supporting wrapping transactions in calls.
2. Place all transactions to [flashbots](https://docs.flashbots.net/), in one bundle.
