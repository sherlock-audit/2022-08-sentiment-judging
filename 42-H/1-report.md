cergyk
# If too much accounts are listed in the Registry, calling view functions could fail by exceeding arbitrary gas limits.

## Summary
RPC calls could fail if the number of accounts gets too large, which could disrupt the monitoring of health factors and ultimately the liquidation process.

## Vulnerability Detail
In the contract Registry, two view functions are used to get the list of all accounts: getAllAccounts() and accountsOwnedBy(address user). Other view functions are used for the keys and the listedLTokens, but only the list of accounts could legitimately reach a significant size.
Even if view functions are gas free, limits can be set by public rpc providers to avoid DoS attacks. For example, calling getAllAccounts() for 25000 accounts consumes 70,576,600 gas (test made using Foundry) which is more than twice the current block limit of Ethereum (30M).

## Impact

The impact would be that, once the number of accounts in the contract Registry reaches a given amount, monitoring tools using these two functions may stop to work correctly, impacting a correct visibility over the protocol. For example, a maintainer could have more difficulties to detect and execute liquidation opportunities.

## Code Snippet

https://github.com/sherlock-audit/2022-08-sentiment-validydy/blob/2123357e2a9866bd62d8fe731b222f917a062d59/protocol/src/core/Registry.sol#L159

https://github.com/sherlock-audit/2022-08-sentiment-validydy/blob/2123357e2a9866bd62d8fe731b222f917a062d59/protocol/src/core/Registry.sol#L176

## Tool used

Manual Review and Foundry to test the gas used by these functions, after pre-populating the array accounts with 25000 addresses.

## Recommendation

As is not a good idea to restrict the max number of accounts, consider using paginated views instead of simply returning the whole list of accounts.