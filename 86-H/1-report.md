cergyk
# All Asset may be locked due to zero adress.

## Summary

zero address in solidity hasa special consideration. zero address become significant because state variables or local variables have zero value by default. zero addess used as burn adresses because the private key corresponding to this zero adress is not known so when the token sent to this adress can made token inaccessable.

## Vulnerability Detail

when the sender send any token in sweepTo() and the sender made mistakes by not check the address to send asset to. it makes all assets will be locked.

## Impact

all asset may be locked and inaccessable when asset sent to zero address.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-calmkidd/blob/5d8b0126f4e4dcdab99c5c89922d74faa99cecb3/protocol/src/core/Account.sol#L163-L174

## Tool used

Manual Review

## Recommendation
add zero address check for validation

import {Errors} from "../utils/Errors.sol";

if (toAddress == address(0)) revert Errors.ZeroAddress();
