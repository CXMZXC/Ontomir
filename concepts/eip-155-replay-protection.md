# EIP-155 Replay Protection

> EIP-155 is an Ethereum Improvement Proposal that introduced replay protection by including chain ID information in signed transaction data. This prevents a signed transaction from being valid on multiple networks, protecting users from replay attacks.

### Background <a href="#background" id="background"></a>

**Replay attacks** occur when a signed transaction valid on one network can be replayed on another network without the user's consent. This is particularly problematic for Ethereum-based networks because:

1. **Address Format**: Ethereum uses hexadecimal addresses (e.g., `0x123...`) that are identical across all EVM chains
2. **Transaction Format**: The same transaction structure and signing scheme across networks
3. **Network Forks**: When chains fork, the same addresses exist on both chains with potentially different balances

EIP-155 solves this by incorporating the chain ID into the transaction signing process, making signatures network-specific.

### How Ontomir EVM Handles Replay Protection <a href="#how-ontomir-evm-handles-replay-protection" id="how-ontomir-evm-handles-replay-protection"></a>

#### Dual Protection System <a href="#dual-protection-system" id="dual-protection-system"></a>

Ontomir EVM implements replay protection at two levels:

\*\*Inherent Protection\*\*: Uses Bech32 addresses with chain-specific prefixes (e.g., \`Ontomir1...\`, \`osmo1...\`). Addresses from one chain are invalid on others, providing automatic replay protection. \*\*EIP-155 Protection\*\*: Requires chain ID in transaction signatures. The EVM module enforces this by default, rejecting transactions without proper chain ID.

#### Chain ID Configuration <a href="#chain-id-configuration" id="chain-id-configuration"></a>

As documented in the Chain ID page, Ontomir EVM uses:

* **EVM Chain ID** (integer): Used for EIP-155 replay protection in Ethereum transactions
* **Ontomir Chain ID** (string): Used for Ontomir SDK transactions with Bech32 addresses

The EVM Chain ID must be unique across all EVM-compatible chains to ensure replay protection. Check \[chainlist.org]\(https://chainlist.org/) before selecting your chain ID.

### Configuring Replay Protection <a href="#configuring-replay-protection" id="configuring-replay-protection"></a>

By default, replay protection is **enabled** on all Ontomir EVM nodes. The system enforces EIP-155 signed transactions, rejecting any unprotected transactions.

#### Allowing Unprotected Transactions (Not Recommended) <a href="#allowing-unprotected-transactions-not-recommended" id="allowing-unprotected-transactions-not-recommended"></a>

Disabling replay protection is \*\*strongly discouraged\*\* as it exposes users to replay attacks. Only consider this for specific testing scenarios or migration purposes.

If absolutely necessary, unprotected transactions can be allowed through a two-step process:

**Step 1: Global Module Parameter**

The EVM module parameter must be changed via governance proposal or genesis configuration:

```
// In x/vm/types/params.go
DefaultAllowUnprotectedTxs = false  // Default value

// To enable in genesis.json
{
  "vm": {
    "params": {
      "allow_unprotected_txs": true  // Must be set to true
    }
  }
}
```

**Step 2: Node Configuration**

Even when globally enabled, each node operator must individually opt-in by modifying their node configuration:

```
# In $HOME/.evmd/config/config.toml
[json-rpc]
# AllowUnprotectedTxs restricts unprotected (non EIP-155 signed) transactions
# to be submitted via the node's RPC when the global parameter is disabled.
allow-unprotected-txs = true  # false by default
```

Both conditions must be met for unprotected transactions to be accepted:

1. Global parameter `allow_unprotected_txs` = true
2. Node configuration `allow-unprotected-txs` = true

### Transaction Types and Protection <a href="#transaction-types-and-protection" id="transaction-types-and-protection"></a>

#### Protected Transactions (EIP-155) <a href="#protected-transactions-eip-155" id="protected-transactions-eip-155"></a>

All modern Ethereum transaction types include chain ID:

```
// EIP-1559 Dynamic Fee Transaction
const tx = {
  type: 2,
  chainId: 9000,  // EVM Chain ID included
  nonce: 0,
  maxFeePerGas: ...,
  maxPriorityFeePerGas: ...,
  // ... other fields
}
```

#### Legacy Unprotected Transactions <a href="#legacy-unprotected-transactions" id="legacy-unprotected-transactions"></a>

Pre-EIP-155 transactions without chain ID:

```
// Legacy transaction without chain ID (rejected by default)
const tx = {
  nonce: 0,
  gasPrice: ...,
  gasLimit: ...,
  to: ...,
  value: ...,
  data: ...,
  // No chainId field
}
```

### Security Implications <a href="#security-implications" id="security-implications"></a>

#### With Replay Protection Enabled (Default) <a href="#with-replay-protection-enabled-default" id="with-replay-protection-enabled-default"></a>

**Protected from:**

* Cross-chain replay attacks
* Accidental transaction execution on wrong networks
* Fork-related replay vulnerabilities

#### With Replay Protection Disabled <a href="#with-replay-protection-disabled" id="with-replay-protection-disabled"></a>

**Vulnerable to:**

* Transactions being replayed on any EVM chain
* Loss of funds through replay attacks
* Unintended contract interactions on multiple chains

### Best Practices <a href="#best-practices" id="best-practices"></a>

1. **Always use EIP-155**: Ensure your wallet and tools sign transactions with chain ID
2. **Verify chain ID**: Double-check the chain ID in your transaction before signing
3. **Keep protection enabled**: Never disable replay protection in production
4. **Monitor transactions**: Track transaction execution across chains during migrations

### Verification <a href="#verification" id="verification"></a>

To verify replay protection status:

```
# Check global parameter
evmd q vm params | grep allow_unprotected_txs

# Check node configuration
grep allow-unprotected-txs $HOME/.evmd/config/config.toml
```

* [EIP-155 Specification](https://eips.ethereum.org/EIPS/eip-155)
* Chain ID Configuration
* [Transaction Types](https://about/docs/evm/next/documentation/concepts/transactions#ethereum-tx-type)
* [EVM Module Parameters](https://about/docs/evm/next/documentation/Ontomir-sdk/modules/vm#parameters)
