csanuragjain
# Account Closing and liquidation will always fail

## Summary
If even a single token linked to account has frozen transfers (due to hack or bad token) then user wont be able to close the account or perform liquidation 

## Vulnerability Detail

1. User account A has 3 tokens T1,T2,T3
2. User has no debt currently and decides to close this account
3. Due to a hack, T2 token suspended all transfers
4. User calls [closeAccount](https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L113) in order to close this account which internally calls account.sweepTo

```
function closeAccount(address _account) public onlyOwner(_account) {
        ...
        account.sweepTo(msg.sender);
        emit AccountClosed(_account, msg.sender);
    }
```

5. Now the problem with sweepTo is even if a single transfer fails the whole function will fail (using safeTransfer)

```
function sweepTo(address toAddress) external accountManagerOnly {
        uint assetsLen = assets.length;
        for(uint i; i < assetsLen; ++i) {
            assets[i].safeTransfer(
                toAddress,
                assets[i].balanceOf(address(this))
            );
            hasAsset[assets[i]] = false;
        }
        delete assets;
        toAddress.safeTransferEth(address(this).balance);
    }
```

6. In our case since T2 has frozen transfers so assets[i].safeTransfer fails for T2 and hence closeAccount fails. Imagine a situation where a token has permanently frozen transfer then this situation becomes permanent

7. This also applies to liquidation which uses sweepTo. In this case also T2 will cause whole liquidation to fail.

## Impact
A genuine request to close user account or to liquidate a unhealthy account will fail due to one bad token

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/Account.sol#L163
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L384
https://github.com/sherlock-audit/2022-08-sentiment-csanuragjain/blob/main/protocol/src/core/AccountManager.sol#L121

## Tool used
Manual Review

## Recommendation
Create an emergency function in Account which uses transfer instead of safeTransfer and will complete even if transfer fails