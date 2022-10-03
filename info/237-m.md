chainNue
# Custom Ownable contract prone to risk availability of protocol

## Summary
Sentiment use a custom `Ownable.sol` contract. This contract contains some modifier and implement function for transfering ownership. It also being used as inheritance to other contracts.
- controller/src/core/ControllerFacade.sol
- oracle/src/chainlink/ChainlinkOracle.sol
- oracle/src/core/OracleFacade.sol
- oracle/src/uniswap/UniV3TWAPOracle.sol
- protocol/src/core/Registry.sol
- protocol/src/core/RiskEngine.sol
- protocol/src/interface/tokens/ILToken.sol
- protocol/src/proxy/Beacon.sol
- protocol/src/utilsPausable.sol

The custom `Ownable.sol` contract doesn't follow transfer-accept ownership pattern or two-step process thus subject to potential crucial error, availability of protocol.

## Vulnerability Detail
An Ownable contract is a standard way of owning a contract. But there is a potential issue within this custom Ownable contract, it allows for the transfer of ownership without validating that the address is a valid address in control of some expected recipient. If this function is used incorrectly, mistype, or any unexpected input, the admin user might be lost and potentially locked up for future usage.

This is categorised as a medium severity, because this could impact availability of protocol.

## Impact
Availability of protocol.

## Recommendation
Consider implementing a transfer-accept ownership pattern or two-step process in those contracts when transfering ownership. This allow an owner to accept the transfer insuring that the account is controlled by a valid user.