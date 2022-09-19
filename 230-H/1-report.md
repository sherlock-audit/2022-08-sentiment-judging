devtooligan
# Protocol still operational and at risk during pause - HIGH

## Summary
A pausable feature has been added to the protocol but it is applied inconsistently to functions, leaving the protocol at risk.

## Vulnerability Detail
`AccountManager` and `LToken` inherit `Pausable.sol` (similar to [OZ Pausable.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/Pausable.sol)) and the `whenNotPaused` modifier has been applied to `deposit`, `depositEth`, and `borrow` in `AccountManager.sol` and `lendTo` in `LToken.sol`.  

A major use for pause functionality is to mitigate risk when a vulnerability is discovered which may involve an expected or ongoing hack.  It is a type of protection against unforeseen future events and it is not known in advance which functions would be appropriate to pause.

One example would be the discovery of a vulnerability in one of the approved tokens used as collateral in which any sort of interaction with the token may put the protocol at risk.  Another example would be the discovery of an exploit related to internal accounting, and if withdraws are not paused, attackers may drain the protocol.

## Impact
High - Depending on the situation, the stakes could be very high, including, but not limited to, draining of protocol funds.

## Tool used
Manual Review

## Recommendation
It is recommended that the `whenNotPaused` modifier be applied to all external functions that could interact directly with any external contracts including `closeAccount()`, `repay()`, `withdraw()`, `withdrawEth()`, `liquidate()`, `settle()`, `exec()`, and `approve()`. 
