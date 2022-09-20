Czar102
# Multiple cross-contract reentrancy vulnerabilities in AccountManager

## Summary
There are multiple reentracny possibilities in `Account` by interacting through `AccountManager`. There is no final check for collateral so that if funds are being transferred out and the attacker gains execution, they can invoke withdrawal again causing a typical cross-contract reentrancy issue. There is no final check for the amount, so that such call doesn't revert.

## Vulnerability Detail
Functions `withdraw, `closeAccount` and `liquidate` invoke withdrawals of tokens without checking whether an account is healthy after all transfers. Let's consider a following scenario:

Alice deposited $1000 of ETH. She borrows the maximum the protocol allowed her, which is $1000 of ETH and $1000 of a reentrant token. Then the price of ETH marginally falls, making her position prone to liquidation. She liquidates her account, but when she pays back the ERC20 token debt, she invokes borrow again and has $1000 of the token back in her account. Then, all tokens are withdrawn, which are worth $2000. Alice exploited the system. Note that the tokens could also be collateral and another, not withdrawn collateral could be then borrowed.

## Impact
An attacker can use reentrancy to drain all borrowable funds, provided there is a single reentrant (e.g. ERC777) token that can be used as collateral or can be borrowed. 

## Code Snippet

```js
function _liquidate(address _account) internal {
	IAccount account = IAccount(_account);
	address[] memory accountBorrows = account.getBorrows();
	uint borrowLen = accountBorrows.length;

	ILToken LToken;
	uint amt;

	for(uint i; i < borrowLen; ++i) {
		address token = accountBorrows[i];
		LToken = ILToken(registry.LTokenFor(token));
		LToken.updateState();
		amt = LToken.getBorrowBalance(_account);
		token.safeTransferFrom(msg.sender, address(LToken), amt);
		LToken.collectFrom(_account, amt);
		account.removeBorrow(token);
	}
	account.sweepTo(msg.sender);
}
```
```js
/**
	@notice Closes a specified account for a user
	@dev Account can only be closed when the account has no debt
		Emits AccountClosed(account, owner) event
	@param _account Address of account to be closed
*/
function closeAccount(address _account) public onlyOwner(_account) {
	IAccount account = IAccount(_account);
	if (account.activationBlock() == block.number)
		revert Errors.AccountDeactivationFailure();
	if (!account.hasNoDebt()) revert Errors.OutstandingDebt();
	account.deactivate();
	registry.closeAccount(_account);
	inactiveAccountsOf[msg.sender].push(_account);
	account.sweepTo(msg.sender);
	emit AccountClosed(_account, msg.sender);
}
```
```js
/**
	@notice Transfers a specified amount of token from the account
		to the owner of the account
	@dev Amount of token can only be withdrawn if the account remains healthy
		after withdrawal
	@param account Address of account
	@param token Address of token
	@param amt Amount of token to withdraw
*/
function withdraw(address account, address token, uint amt)
	external
	onlyOwner(account)
{
	if (!riskEngine.isWithdrawAllowed(account, token, amt))
		revert Errors.RiskThresholdBreached();
	account.withdraw(msg.sender, token, amt);
	if (token.balanceOf(account) == 0)
		IAccount(account).removeAsset(token);
}
```

## Tool used

Manual Review

## Recommendation

Add `ReentrancyGuard` to all listed functions and `borrow`.
