# Precompiled Contracts

> A hub for precompiled contracts that bridge EVM and Ontomir SDK modules.

### Precompiled Contracts <a href="#precompiled-contracts-1" id="precompiled-contracts-1"></a>

Precompiled contracts provide direct, gas-efficient access to native Ontomir SDK functionality from within the EVM. Build powerful hybrid applications that leverage the best of both worlds. This page provides an overview of the available precompiled contracts, each with detailed documentation on its usage, Solidity interface, and ABI.

\*\*Critical: Understanding Token Decimals in Precompiles\*\*

All Ontomir EVM precompile contracts use the **native chain's decimal precision**, not Ethereum's standard 18 decimals (wei).

**Why this matters:**

* Ontomir chains typically use 6 decimals, while Ethereum uses 18
* Passing values while assuming conventional Ethereum exponents could cause significant miscalculations
* Example: On a 6-decimal chain, passing `1000000000000000000` (1 token in wei) will be interpreted as **1 trillion tokens** **Before using any precompile:**

1. Check your chain's native token decimal precision
2. Convert amounts to match the native token's format
3.  Never assume 18-decimal precision

    ```
    // WRONG: Assuming Ethereum's 18 decimals
    uint256 amount = 1 ether; // 1000000000000000000
    staking.delegate(validator, amount); // Delegates 1 trillion tokens!

    // CORRECT: Using chain's native 6 decimals
    uint256 amount = 1000000; // 1 token with 6 decimals
    staking.delegate(validator, amount); // Delegates 1 token
    ```

### Available Precompiles <a href="#available-precompiles" id="available-precompiles"></a>

Access native Ontomir SDK features directly from Solidity:

Interface with native Ontomir SDK tokens for balance and supply queries. Explore Bank → Convert addresses between Ethereum hex and Ontomir bech32 formats. Explore Bech32 → Handle IBC packet lifecycle callbacks in smart contracts. Explore Callbacks → Withdraw staking rewards and interact with the community pool. Explore Distribution → Utilize standard ERC20 token functionality for native Ontomir tokens. Explore ERC20 → Participate in on-chain governance through proposals and voting. Explore Governance → Perform cross-chain token transfers via the IBC protocol. Explore ICS20 → Execute P-256 elliptic curve cryptographic operations. Explore P256 → Manage validator slashing and jailing for network security. Explore Slashing → Perform validator operations, manage delegations, and handle staking. Explore Staking → Wrap native tokens to provide ERC20-compatible functionality. Explore WERC20 →
