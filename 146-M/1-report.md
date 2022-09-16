Kumpa
# User can deposit unallowed Collateral to his account by using ```AccountManager.exec()``` 

Admin is able to restrict certain collateral by calling ```toggleCollateralStatus()``` which stop users from depositing targeted token into his account. The user however is able to bypass this restriction by calling ```AccountManager.exec()``` and swap one of the tokens with the restricted one from the controller. 

### Proof of Concept 
1.Admin ```toggleCollateralStatus()``` to prevent users from adding token1 into their accounts

2.Users seek another way to add that restricted token into his account by calling ```AccountManager.exec()``` and add data to swap token with restricted token 

### Mitigations 
Add a check (CollateralAllowed[token]) in ```AccountManager.exec()```

