ladboy233
# Proxy initialization is subject to front running.

## Summary

The protocol use proxy pattern so it is requested that the function

```
 function init()
```

is properly called after the deployment, otherwise, malicious user can call the init before the project, 
which makes the deployed smart contract not usable.

## Vulnerability Detail

a proxy can be upgraded and initialized by a malicious front running if more than one transaction is used to upgrade or deploy.

## Impact

Waste of deployment gas. malicious smart contract ownership takeover.

## Code Snippet

## Tool used

Manual Review

## Recommendation

We recommend adding a function to the proxy that allows the implementation address
to be modified and then executed a function on the new implementation address to avoid front running.
