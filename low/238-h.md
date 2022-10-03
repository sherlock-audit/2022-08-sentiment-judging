Waze
# Missing check zero address may lead ether freeze in safeTransferETH .

## Summary
According to the Solidity docs), "The low-level functions call, delegatecall and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed".
## Vulnerability Detail
The safeTransferETH() will not notice anything went wrong. This may result in user funds lost because funds were transferred. And the safeTransferETH() fails but doesn't revert due to non existance contract address.
## Impact
When the contract's destination existence is not verified (to), the call will fails but returns success due to nonexistance contract and we receive nothing on return.
## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-calmkidd/blob/5d8b0126f4e4dcdab99c5c89922d74faa99cecb3/protocol/src/utils/Helpers.sol#L35-L38
## Tool used

Manual Review

## Recommendation
Check for contract destination is existence on low-level calls, so that failures are not missed. 
e.x
add this:
import {Errors} from "../utils/Errors.sol";
if (to == address(0)) revert Errors.ZeroAddress();

or

require( to != address(0),"zero address"); 