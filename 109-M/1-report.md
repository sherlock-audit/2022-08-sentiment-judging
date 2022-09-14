0xNazgul
# [NAZ-M2] Protocol Admin is a Single Point of Failure

## Summary
Protocol admin can arbitrarily and unilaterally update implementations, critical addresses, configure critical parameters and pause/unpause. This presents a critical single point of failure. 

## Vulnerability Detail
Sentiment protocol uses a custom `Ownable.sol` to implement its single admin. If a protocol admin becomes malicious or compromised, the entire protocol is immediately at risk for all existing/future markets/participants. 

## Impact
While the updation of many critical parameters emit events, that only lets market participants react after the fact because these changes are not time-delayed.

## Code Snippet
[`AccountManager.sol#L76`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/AccountManager.sol#L76), [`AccountManager.sol#L395`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/AccountManager.sol#L395), [`Registry.sol#L74`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/Registry.sol#L74), [`Registry.sol#L95`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/Registry.sol#L95), [`RiskEngine.sol#L57`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/core/RiskEngine.sol#L57), [`Beacon.sol#L18`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/proxy/Beacon.sol#L18), [`BeaconProxy.sol#L21`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/proxy/BeaconProxy.sol#L21), [`Proxy.sol#L21`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/proxy/Proxy.sol#L21), [`Proxy.sol#L25`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/proxy/Proxy.sol#L25), [`LToken.sol#L116`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/tokens/LToken.sol#L116)

## Tool used
Manual Review

## Recommendation
The protocol admin should nevertheless be a reasonable threshold multisig (e.g. 4/7, 5/9) with diverse owners and (cold/hardware) wallets until it is backed by token-holder governance, i.e., it should certainly never be an EOA. The highest possible operational security measures should be taken for all multisig owners and wallets. The assignment of roles to and management by different addresses should be enforced at the earliest in the spirit of the Principle of Least Privilege and Principle of Separation of Privilege.