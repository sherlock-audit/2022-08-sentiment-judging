TomJ
# No upper bounds on how many different asset/borrow types user can own may cause liquadate() to fail

## Summary
There are no upper bounds on how many different asset (collateral) types and borrow types user can own in their account.
Since gas requirements can change over time, if user own large amount of asset types and borrow types, it may lead to `liquidate()` of `AccountManager.sol` failing even though user's account is unhealthy, since it will revert due to **out of gas error**.
Also there is no 0 amount check on deposit() so it is possible for user to add assets (collateral types) to their account
with 0 amount which makes it easier for an user to gain a lot of asset types.

## Vulnerability Detail
Since `liquidate()` has a lot of loops implemented as it is described in below code snippet, 
user can cause this function to cause an out of gas error by owning enormous number of asset and borrow types.

## Impact
User can block maintainers from executing liquidate() even though user's account is unhealthy by owning enormous amount of asset and borrow types.

## Code Snippet
There are no upper bounds on the number of asset/borrow types being transferred in following loops within [liquidate()](https://github.com/sherlock-audit/2022-08-sentiment-TomJ-BB/blob/82066aae2d33a0f9ecbd57fea0f11f5579613df8/protocol/src/core/AccountManager.sol#L250-L255):

liquadate() -> isAccountHealthy() -> [_getBalance()](https://github.com/sherlock-audit/2022-08-sentiment-TomJ-BB/blob/82066aae2d33a0f9ecbd57fea0f11f5579613df8/protocol/src/core/RiskEngine.sol#L154-L158)
```solidity
RiskEngine.sol
150:    function _getBalance(address account) internal view returns (uint) {
151:        address[] memory assets = IAccount(account).getAssets();
152:        uint assetsLen = assets.length;
153:        uint totalBalance;
154:        for(uint i; i < assetsLen; ++i) {
155:            totalBalance += _valueInWei(
156:                assets[i],
157:                IERC20(assets[i]).balanceOf(account)
158:            );
159:        }
160:        return totalBalance + account.balance;
161:    }
```

liquadate() -> isAccountHealthy() -> [_getBorrows()](https://github.com/sherlock-audit/2022-08-sentiment-TomJ-BB/blob/82066aae2d33a0f9ecbd57fea0f11f5579613df8/protocol/src/core/RiskEngine.sol#L168-L173)
```solidity
RiskEngine.sol
163:    function _getBorrows(address account) internal view returns (uint) {
164:        if (IAccount(account).hasNoDebt()) return 0;
165:        address[] memory borrows = IAccount(account).getBorrows();
166:        uint borrowsLen = borrows.length;
167:        uint totalBorrows;
168:        for(uint i; i < borrowsLen; ++i) {
169:            address LTokenAddr = registry.LTokenFor(borrows[i]);
170:            totalBorrows += _valueInWei(
171:                borrows[i],
172:                ILToken(LTokenAddr).getBorrowBalance(account)
173:            );
174:        }
175:        return totalBorrows;
176:    }
```

liquadate() -> [_liquidate()](https://github.com/sherlock-audit/2022-08-sentiment-TomJ-BB/blob/82066aae2d33a0f9ecbd57fea0f11f5579613df8/protocol/src/core/AccountManager.sol#L375-L383)
```solidity
AccountManager.sol
375:        for(uint i; i < borrowLen; ++i) {
376:            address token = accountBorrows[i];
377:            LToken = ILToken(registry.LTokenFor(token));
378:            LToken.updateState();
379:            amt = LToken.getBorrowBalance(_account);
380:            token.safeTransferFrom(msg.sender, address(LToken), amt);
381:            LToken.collectFrom(_account, amt);
382:            account.removeBorrow(token);
383:        }
```

liquadate() -> _liquidate() -> [sweepTo()](https://github.com/sherlock-audit/2022-08-sentiment-TomJ-BB/blob/82066aae2d33a0f9ecbd57fea0f11f5579613df8/protocol/src/core/Account.sol#L165-L171)
```solidity
Account.sol
165:        for(uint i; i < assetsLen; ++i) {
166:            assets[i].safeTransfer(
167:                toAddress,
168:                assets[i].balanceOf(address(this))
169:            );
170:            hasAsset[assets[i]] = false;
171:        }
```

## Tool used
Manual Review

## Recommendation
Add upper bound on how much asset and borrow types user can own (example: limit of 10 types each).
This can be done in client-side or off-chain but only relying on this is not recommended, 
since it is possible for a malicious user to bypass client-side by directly accessing the contract.
Therefore it is important to implement upper bound limit within the contract logic.