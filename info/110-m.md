0xNazgul
# [NAZ-M3] Incorrect Implementation address emitted.

## Summary
An event emits an incorrect address that could lead to issues and confusion in the future.

## Vulnerability Detail
The `Upgraded` event emitted in `upgradeTo()` uses an incorrect implementation address. The `Upgraded` event in `upgradeTo()` is emitted when an implementation is upgraded from the beacon contract and its parameter indicates the address to which the implementation address is upgraded to. The `Upgraded` event emitted uses `newImplementation` instead of the indcated origanal implementation. 

## Impact
This may mislead protocol user interfaces and off-chain monitoring systems to misinterpret the upgraded implementation causing confusion, flagging of alerts or DoS.

## Code Snippet
[`Beacon.sol#L11`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/proxy/Beacon.sol#L11)

## Tool used
Manual Review

## Recommendation
Change `event Upgraded(address indexed implementation);` To `event Upgraded(address indexed newImplementation);` as is done in [`Proxy.sol#L14`](https://github.com/sherlock-audit/2022-08-sentiment-0xNazgul/blob/main/protocol/src/proxy/Proxy.sol#L14).