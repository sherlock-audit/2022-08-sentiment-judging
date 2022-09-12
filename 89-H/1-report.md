Czar102
# Quadratic gas usage can cause DoS of liquidation

## Summary
The `_liquidate` function has a quadratic gas usage as a function of the number of borrows, which may allow the attacker to DoS the function not to be liquidated and since exploit the system.
## Vulnerability Detail
The `_liquidate` function calls `account`'s `removeBorrow` with consecutive tokens from the query made before the loop. Since, in the first `removeBorrow` the first token will be deleted and the last will be put in its place, in the second the second will be dealt with like so with it and so on. Thus, the number of steps in the `removeBorrow` function will rise to up to `borrowLen / 2` and then fall linearly to zero, thus a quadratic number of storage reads is executed, which is expensive in terms of gas.

Additionally, there are no constraints on the value borrowed which makes it possible to make it economically nonviable to liquidate an account with low values due to gas fees, which tend to be high when market is volatile.

## Impact
Since quadratic gas usage, it may be the case that liquidation will be very gas-consuming or even won't fit in a block which causes no economic incentive to liquidate an account or no possibility of such action, breaking the entire system.
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
function removeBorrow(address token) external accountManagerOnly {
	_remove(borrows, token);
}

function _remove(address[] storage arr, address token) internal {
	uint len = arr.length;
	for(uint i; i < len; ++i) {
		if (arr[i] == token) {
			arr[i] = arr[arr.length - 1];
			arr.pop();
			break;
		}
	}
}
```

## Tool used

Manual Review

## Recommendation
It is possible to construct sets with addition and removal in constant gas thanks to structures such as OpenZeppelin's [EnumerableSet](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol).

Don't allow for small loans to make liquidations affordable in terms of gas cost.
