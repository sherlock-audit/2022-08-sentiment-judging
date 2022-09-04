JohnSmith
# Can not create LToken vaults with assets that do not conform to `IERC20Metadata`

https://github.com/sentimentxyz/protocol/blob/4e45871e4540df0f189f6c89deb8d34f24930120/src/tokens/utils/ERC4626.sol#L41
## [M] Can not create LToken vaults with assets that do not conform to `IERC20Metadata`
Some ERC20 tokens do not conform to `IERC20Metadata`, because it is OPTIONAL to do so, and as result call to `decimals()` may result in revert.
### Proof of Concept
```solidity
protocol/src/tokens/utils/ERC4626.sol
35:     function initERC4626(
36:         ERC20 _asset,
37:         string memory _name,
38:         string memory _symbol
39:     ) internal {
40:         asset = _asset;
41:         initERC20(_name, _symbol, asset.decimals()); //@audit asset.decimals() may revert
42:     }
```
### Mitigation
We can mitigate this by doing something like OpenZeppelin did:
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol#L36-L41
```diff
function initERC4626(
        ERC20 _asset,
        string memory _name,
        string memory _symbol
    ) internal {
        asset = _asset;
-       initERC20(_name, _symbol, asset.decimals());
+	uint8 decimals_;
+	try IERC20Metadata(address(asset_)).decimals() returns (uint8 value) {
+	decimals_ = value;
+	} catch {
+		decimals_ = 18;
+	}
+	initERC20(_name, _symbol, decimals_);
    }
```
