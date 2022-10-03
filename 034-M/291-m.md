0xc0ffEE
# can not repay native token

https://github.com/sherlock-audit/2022-08-sentiment-thongtrungtran/blob/37f9955a439f5c3c3f8b1d805460aef8a6b9ba7e/protocol/src/core/AccountManager.sol#L324

https://github.com/sherlock-audit/2022-08-sentiment-thongtrungtran/blob/37f9955a439f5c3c3f8b1d805460aef8a6b9ba7e/protocol/src/core/AccountManager.sol#L236

The repaying of native token will fails because in `repay` function, the `withdraw` is executed with `token = address(0)` which is away fails since it is calling to non-contract address