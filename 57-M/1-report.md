grhkm
# Denial of Service by well funded first user

## [M] Denial of Service by well funded first user
### Problem
First user can manipulate share price by sending tokens to the system externally.
They can setup a front-running got which will deposit higher amount than legit users, preventing them to buy a single share.
### Proof of Concept
- Alice buys first share for 1 wei. Price of 1 share becomes 1 wei;
- Alice runs a front-running bot, which reads mempool;
- Bob deposits arbitrary amount, i.e. 100 ether;
- Alice's bot sends transaction with higher fee to ERC.transfer(address,uint) to LToken vault 100 ether or more
- Share price becomes 100 ether + 1wei or more;
- Bob's transaction reverts because his amount sent is not enough to buy a single share;
- Alice or her bot then can redeem assets;
- As result nobody can deposit, and nothing to borrow;

When Alice pays for gas, you pay for reputation damage.
### Mitigation
- Use Flashbots to avoid mempool.
- Force users to deposit at least some amount in the LToken vault (Uniswap forces users to pay at least 1e18).
That way the amount the attacker will need to ERC20.transfer to the system will be at least X * 1e18 instead of X which is unrealistic.