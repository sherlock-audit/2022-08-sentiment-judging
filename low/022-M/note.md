## Sherlock Team

I believe the `require(0 != address(target).code.length)` check can be added but as the targets are admin protected through `ControllerFacade.sol` it's not a medium or high vulnerability.

`053-h.md` mentions signature hash collission as an attack, I don't think that applies as whitelisted contracts are used. As far as I know there is no way for a non-admin user to whitelist a contract or whitelist other functions.

