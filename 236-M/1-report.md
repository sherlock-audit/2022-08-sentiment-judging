devtooligan
# Proxy based on pre-2018 antipattern misses important safety precaution - MEDIUM

## Summary

`Proxy.sol` lacks a safety feature preventing admins from inadvertently overwriting proxy storage through function collisions, potentially bricking the protocol.

## Vulnerability Detail
The `Proxy.sol` contract which is used as the proxy contract with `Registry`, `AccountManager`, `LERC20`, `LEther` uses an antiquated proxy pattern that [Open Zeppelin replaced with the Transparent Proxy Pattern (TPP) in 2018](https://blog.openzeppelin.com/the-transparent-proxy-pattern/).  The old proxy is considered an antipattern due to the question of how to proceed when a proxy function selector matches that of an implementation contract function either because of the same name or via function selector collision.  As recently 08-Sep-2022, security researcher SamCZSun [commented on Twitter](https://twitter.com/samczsun/status/1568058908882931712?s=20&t=AULQN3WtKj6MS4Mkez7aCw) that function collisions occur much more often than previously thought.  

## Code Snippet
The key difference between the proxy currently being used and [OZ's current implementation of TPP](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol) is that admin calls which match `adminOnly` functions are immediately executed by the proxy contract logic and never get forewarded to the implementation contract whereas 100% of calls made by non-admin users do get forewarded via `delegatecall`.

```diff
diff --git a/protocol/src/proxy/BaseProxy.sol b/protocol/src/proxy/BaseProxy.sol
index a2c5588..9954118 100644
--- a/protocol/src/proxy/BaseProxy.sol
+++ b/protocol/src/proxy/BaseProxy.sol
@@ -12,8 +12,11 @@ abstract contract BaseProxy {
     event AdminChanged(address previousAdmin, address newAdmin);

     modifier adminOnly() {
-        if (msg.sender != getAdmin()) revert Errors.AdminOnly();
-        _;
+        if (msg.sender == getAdmin()) {
+            _;
+        } else {
+            _delegate(getImplementation());
+        }
     }
```

## Tool used

Manual Review

## Recommendation
Bring the proxy up to modern spec by implementing the suggested code change.  Note: There is one downside to implementing this which is that an admin will never be able to call a function with a matching selector on the implementation contract.
