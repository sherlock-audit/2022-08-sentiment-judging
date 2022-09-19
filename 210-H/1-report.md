devtooligan
# Use a two-step procedure for non-recoverable critical operations  - HIGH

## Summary
Several critical operations are done in one function call.  This schema is error-prone and can lead to irrecoverable mistakes.

## Vulnerability Detail
This vuln applies to multiple non-recoverable functions throughout the protocol, but most critically it is seen in `changeAdmin` function in [the BaseProxy contract](https://github.com/sherlock-audit/2022-08-sentiment-devtooligan/blob/c03672aa8f08d6fba825060715c93e6169f5ad11/protocol/src/proxy/BaseProxy.sol#L19) which is inherited by `Proxy.sol` and used with implementation contracts: `Registry`, `AccountManager`, `LEther`, and `LToken` as well as `BeaconProxy.sol` which is used with `Account`.  

The code used by `BaseProxy.sol` is based on code found in OpenZeppelin's antiquated `Ownable.sol` which is now considered anti-pattern due to security concerns.  The industry standard is quickly moving towards two-step ownable as evidenced by numerous protocol updates as well as OpenZeppelin's recent merge of [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) released just 17 days ago as of the writing of this report.

Due to an admin being fooled, an admin acting maliciously, or admin mistake, a new admin could be set to the address of an attacker resulting in the immediate loss of funds, or to an unknown address, resulting in the loss of upgradeability.

## Impact
HIGH - Security research firms such as Trail of Bits have [identified similar vulnerabilities as a High impact vulnerability](https://github.com/trailofbits/publications/blob/master/reviews/hermez.pdf) and in this case due to the extensive use of proxies, this vulnerability could result in the complete draining of the protocol at very little cost to the attacker.

## Code Snippet
```
//  👎
    function _setAdmin(address admin) internal {
        if (admin == address(0)) revert Errors.ZeroAddress();
        emit AdminChanged(getAdmin(), admin);
        StorageSlot.setAddressAt(_ADMIN_SLOT, admin);
    }

//  👍 

    function _changeAdmin(address newPendingAdmin) internal virtual override {
         StorageSlot.setAddressAt(_PENDING_ADMIN_SLOT, newPendingAdmin);
    }

    function acceptAdmin() external {
        require(pendingAdmin() == msg.sender);
        StorageSlot.setAddressAt(_ADMIN_SLOT, getPendingAdmin());
    }
```

## Tool used
Manual Review

## Recommendation
At a minimum, implement two-step functionality as shown above with all `setAdmin` functions.  Additionally, since this is not a commonly called function, consider implementing a timelock so that the new admin is not updated immediately.