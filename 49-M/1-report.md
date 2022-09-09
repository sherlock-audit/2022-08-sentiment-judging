minera
# Potential Multichain signature replay in token approval

## Summary

Because we are not using chain id to verify the signature in permit funciton in ERC20.sol, signature can be reused in another blockchain to replay the transaction.

## Vulnerability Detail

Let's look into the permit function in ERC20.sol

```
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual {
        require(deadline >= block.timestamp, "PERMIT_DEADLINE_EXPIRED");

        // Unchecked because the only math done is incrementing
        // the owner's nonce which cannot realistically overflow.
        unchecked {
            address recoveredAddress = ecrecover(
                keccak256(
                    abi.encodePacked(
                        "\x19\x01",
                        DOMAIN_SEPARATOR(),
                        keccak256(
                            abi.encode(
                                keccak256(
                                    "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
                                ),
                                owner,
                                spender,
                                value,
                                nonces[owner]++,
                                deadline
                            )
                        )
                    )
                ),
                v,
                r,
                s
            );

            require(recoveredAddress != address(0) && recoveredAddress == owner, "INVALID_SIGNER");

            allowance[recoveredAddress][spender] = value;
        }

        emit Approval(owner, spender, value);
    }
```

we use the field owner, spender, value, bounces and deadline to verify the signature.

we are missing the chain Id here.

## Impact

because we are not using the chain id to verify the signature, a user can take the signature from one blockchain and replay the transaction using the same signature in another blockchain to get the token approval.

## Code Snippet

## Tool used

Manual Review

## Recommendation

we recommand add chain id to verify the signature.

```
function getChainID() internal view returns (uint256) {
    uint256 id;
    assembly {
        id := chainid()
    }
    return id;
}
```
