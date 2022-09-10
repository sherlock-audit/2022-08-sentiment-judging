cergyk
# No Transfer Ownership Pattern

## Summary

https://github.com/sherlock-audit/2022-08-sentiment-andyfeili/blob/96338b720493bc6dcbfa8ed24b75af53adc7900d/oracle/src/utils/Ownable.sol#L21-L25

## Vulnerability Detail

In `Ownable.sol` the `transferOwnership()` function does not use a two step transfer of ownership pattern.

The current owership transfer process involves the current owner calling `transferOwnership()`.
This function checks the new address is not the zero address and proceeds to write the new address into the `admin` state variable.

If the nominated EOA account is not a valid account, it is entirely possible the owner may accidentally transfer ownership to an uncontrolled account, breaking all functions which require the `adminOnly()` modifier. 

## Recommendation

Implement a two step process where the owner nominates an account and the nominated account needs to call an acceptOwnership() function for the transfer to fully succeed. This ensures the nominated EOA account is a valid and active account.