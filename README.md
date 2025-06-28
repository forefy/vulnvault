# Vulnerable Vault
An intentionally vulnerable smart contract.

## Vulnerability brief

The project consists of a simple token vault.
- Users can deposit funds
- after enough time they can withdraw funds and receive yields based on several factors

```bash
tree
.
├── src/                                    
│   └── Vault.sol             # A vulnerable vault, with a bad accounting logic
│   └── VaultToken.sol        # Generic ERC20 token implementation
├── test/                                    
│   └── Vault.t.sol           # Solution PoC test file
```

One of the yield calculation's deciding factors is the contracts balance.
The problem with that is that the balance, calcualted by `token.balanceOf(address(this))`,
can be manipulated by an attacker sending funds *directly*, i.e. `transfer()` instead of using `deposit()`, to a situation in which funds it sends directly bump the possible yield gains to be more than what is transferred.

This is the vulnerable function:
```javascript
    function calculateYield(address _staker) public view returns (uint256) {
        Stake memory stake = stakes[_staker];
        uint256 stakingDuration = block.timestamp - stake.startTime;
        uint256 scaleFactor = 100000;
        uint256 balanceBasedBonus = (getVaultBalance() * 5) / 100;

        // The longer the stake duration the bigger the yield
        // The higher the stake amount the bigger the yield
        // The more total balance the vault has, the bigger the yield
        uint256 yield = ((stakingDuration * stake.amount * balanceBasedBonus) /
            (totalSupply * scaleFactor)) + balanceBasedBonus;
        return yield;
    }
```

An attacker might realize that the yield calculation can be abused under certain conditions,
and follow these steps:
- Normally deposit some funds in the vault
- Wait a time period (e.g. 6 months) to gain yield
- Before withdrawal, directly transfer tokens to the vault to tamper with it's yield calculation to the attacker's benefit
- Withdraw all invested funds, with fees covering the direct transfer losses, and steal extra tokens due to the tampered yield calculation.


## Required installations:
```
forge install OpenZeppelin/openzeppelin-contracts --no-commit
```

## Run an innocent usecase:
```
forge test -vv --match-test testNormalUse
```

## Run the attack PoC:
```
forge test -vv --match-test testAttack
```