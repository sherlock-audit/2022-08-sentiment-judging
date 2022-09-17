0x52
# Delisted assets can still be deposited and borrowed against by accounts that already have them 

## Summary

Delisting an asset does not prevent accounts, that already contain the asset from depositing more. It blocks the deposit function but users can sidestep but sending tokens directly to their account

## Vulnerability Detail

AccountManger.sol#deposit attempts to block deposits from assets that are not on the list of supported collateral. When calculating the health of the account, the total balance of the account address is considered. An account that already has the asset on it's assets list doesn't need to use AccountManger.sol#deposit because they can transfer the tokens directly to the account contract. This means that these account can continue to add to and use delisted assets as collateral.

## Impact

One of two scenarios depending on actions taken by the protocol. If the asset is delisted from accountManager.sol and the oracle is removed then all users with loans taken against the asset will likely be immediately liquidated, which is highly unfair to users. If the asset is just delisted from accountManager.sol then existing users that already have the asset would be able to continue using the asset to take loans. If the reason an asset is being delisted is to prevent a vulnerability then the exploit would still be able to happen due to these accounts sidestepping deposit restrictions.

## Code Snippet

[AccountManager.sol#L162-L172](https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/core/AccountManager.sol#L162-L172)

## Tool used

Manual Review

## Recommendation

Calculations for account health should be split into two distinct categories. When calculating the health of a position post-action, unsupported assets should not be considered in the total value of the account. When calculating the health of a position for liquidation, all assets on the asset list should be considered. This prevent any new borrowing against a delisted asset but doesn't risk all affected users being liquidated unfairly.
