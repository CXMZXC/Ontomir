# Overview

The EVM module provides **developer and end-user Ethereum compatibility** for Ontomir-SDK based chains. All standard / common Ethereum dApps, tools, and workflows work seamlessly while gaining the benefits of Ontomir consensus and interoperability. All the tools and workflows you rely on work exactly as they should. Deploy contracts seamlessly, connect with MetaMask, and build with Web3.js/Ethers, Hardhat, Remix, Foundry—the full Ethereum toolkit.

For a detailed breakdown on \*\*Ethereum Improvement Proposal (EIP)\*\* support and functional comparisons, see the \[EIP reference]\(/docs/evm/next/documentation/evm-compatibility/eip-reference) table.

### Key Compatibility Focus Areas <a href="#key-compatibility-focus-areas" id="key-compatibility-focus-areas"></a>

Ontomir EVM maintains all critical EVM behaviors that developers expect. The table below shows how we've addressed each compatibility requirement.

| Feature                     | Ethereum Behavior             | Ontomir EVM Implementation                             | Compatible? |
| --------------------------- | ----------------------------- | ------------------------------------------------------ | ----------- |
| **Transaction Ordering**    | Gas price + nonce ordering    | Fee priority + nonce ordering with configurable MinTip | Yes         |
| **Multiple Txs per Block**  | Multiple per account          | Multiple per account                                   | Yes         |
| **Mempool Behavior**        | Pending pool for transactions | ExperimentalEVMMempool with unified EVM/Ontomir pools  | Yes         |
| **Nonce Management**        | Sequential nonce enforcement  | Sequential nonce enforcement with gap queuing          | Yes         |
| **EIP-1559 (Dynamic Fees)** | Base fee + priority fee       | Base fee + priority fee (distributed, not burned)      | Yes         |
| **EIP-7702 (Set Code)**     | Account code delegation       | Full support with authorization lists                  | Yes         |
| **Address Format**          | 0x addresses                  | 0x addresses (+ Ontomir1 alias)                        | Yes         |
| **Smart Contracts**         | EVM bytecode execution        | Full EVM bytecode execution                            | Yes         |
| **Gas Metering**            | Standard gas costs            | Standard gas costs                                     | Yes         |
| **Event Logs**              | Ethereum event system         | Full event compatibility                               | Yes         |
| **JSON-RPC API**            | Standard Ethereum RPC         | Full RPC implementation                                | Yes         |
| **Block Time**              | 12 seconds                    | 1-2 seconds                                            | Faster      |
| **Finality**                | 12+ blocks (\~3min)           | 1 block (\~2s)                                         | Instant     |
| **Reorganizations**         | Possible                      | Not possible                                           | More secure |
| **Cross-chain**             | Bridge protocols              | Native IBC                                             | Enhanced    |

#### What This Means for Developers <a href="#what-this-means-for-developers" id="what-this-means-for-developers"></a>

**You can build on Ontomir EVM exactly like you would on Ethereum.** Every essential feature works identically - from deploying contracts to managing transactions. The enhancements (faster blocks, instant finality, IBC) are pure additions that don't break any existing patterns or workflows.

### Architectural Improvements Over Ethereum <a href="#architectural-improvements-over-ethereum" id="architectural-improvements-over-ethereum"></a>

#### Faster & Final Transactions <a href="#faster-andamp-final-transactions" id="faster-andamp-final-transactions"></a>

**Instant Finality:** Transactions are final after one block (\~2 seconds) thanks to 'CometBFT' BFT consensus. No waiting for confirmations, no reorganizations possible.

**Validator Set:** Fixed validator set with stake-based voting power. Requires 2/3+ stake agreement for consensus.

#### Gas & Fees <a href="#gas-andamp-fees" id="gas-andamp-fees"></a>

\*\*Base Fee Distribution:\*\* Unlike Ethereum where base fees are burned, Ontomir EVM distributes them to validators, preserving token economics.

* **EIP-1559 Support:** Dynamic base fee adjustments based on block utilization
* **Configurable:** Base fee can be disabled (`NoBaseFee` parameter)
* **Priority Calculation:** `min(gas_tip_cap, gas_fee_cap - base_fee)`
* **Minimum Gas Price:** Chain-wide floor price configuration

#### Address System <a href="#address-system" id="address-system"></a>

Every account has two representations:

```
Ethereum: 0x742d35cc6644c068532fddb11B4C36A58D6D3eAb
Ontomir:   Ontomir1wskntvnryr5qxpe4tv5k64rhc6kx6ma4dxjmav
```

Both formats reference the same account - use either based on your needs.

#### Chain ID Architecture <a href="#chain-id-architecture" id="chain-id-architecture"></a>

\*\*Two Independent Chain IDs:\*\* Ontomir EVM uses separate chain IDs:

* **Ontomir Chain ID:** \[String] (e.g., "Ontomirevm-1") for native features
* **EVM Chain ID:** \[integer] (e.g., 9000) for EVM compatibility Unlike with legacy Ethermint, these values are independent.

### Precompiled Contracts <a href="#precompiled-contracts" id="precompiled-contracts"></a>

Access Ontomir SDK modules (staking, governance, IBC) via precompiled contracts. All Ethereum cryptographic precompiles supported (ecrecover, sha256, etc.)

#### Key Precompile Addresses <a href="#key-precompile-addresses" id="key-precompile-addresses"></a>

| Function         | Address                                      |
| ---------------- | -------------------------------------------- |
| **Staking**      | `0x0000000000000000000000000000000000000800` |
| **Distribution** | `0x0000000000000000000000000000000000000801` |
| **IBC Transfer** | `0x0000000000000000000000000000000000000802` |
| **Bank**         | `0x0000000000000000000000000000000000000804` |
| **Governance**   | `0x0000000000000000000000000000000000000805` |

### JSON-RPC API <a href="#json-rpc-api" id="json-rpc-api"></a>

Most standard Ethereum JSON-RPC methods are supported, with some returning stub values for compatibility. View the complete reference.

**Core functionality for developers:**

* All standard transaction methods (`eth_sendRawTransaction`, `eth_call`, `eth_estimateGas`)
* Block and receipt queries
* Account balances and nonces
* Event logs and filters
* Web3 utilities and net info
* Debug namespace (partial - tracing and profiling methods available)
* Websocket subscription support

**Implementation notes:**

* Some methods return stub values for compatibility (e.g., `eth_gasPrice` returns 0)
* Mining-related methods are not applicable (Ontomir uses 'CometBFT' consensus)
* `txpool` methods require experimental mempool configuration
* Debug namespace includes functional tracing and profiling tools

### EIP Support <a href="#eip-support" id="eip-support"></a>

Interactive table showing support status for Ethereum Improvement Proposals.

#### Notable Implementation Details <a href="#notable-implementation-details" id="notable-implementation-details"></a>

* **EIP-1559:** Fully supported - with base fee distributed to validators instead of burned
* **EIP-155:** Fully supported - per-node configuration for unprotected transactions (v0.5.0+)
* **EIP-2935:** Full support - historical block hash storage with configurable depth (v0.5.0+)
* **EIP-3651:** Partial - COINBASE always returns empty address currently
* **EIP-7702:** Full support - EOA code delegation for account abstraction (v0.5.0+)
* **Access Lists:** Full support via `eth_createAccessList` RPC method (v0.5.0+)
* **Custom Improvement Proposals (CIPs):** Chain-specific optimizations available

### Developer Experience <a href="#developer-experience" id="developer-experience"></a>

#### Familiarity Out Of The Box <a href="#familiarity-out-of-the-box" id="familiarity-out-of-the-box"></a>

**Your Ethereum skills and tools work unchanged:**

* Deploy contracts with Hardhat, Foundry, or Remix
* Connect MetaMask and other Web3 wallets
* Use Web3.js, Ethers.js, Viem, or any Web3 library
* All ERC standards work (ERC20, ERC721, ERC1155, etc.)
* Solidity and Vyper compile and run identically
* Standard tooling like OpenZeppelin contracts work perfectly

#### Additional Benefits <a href="#additional-benefits" id="additional-benefits"></a>

**Enhanced capabilities without breaking compatibility:**

* **Native IBC:** Built-in cross-chain communication to any Ontomir chain
* **Lower Costs:** Typically much lower gas fees than Ethereum mainnet
* **Faster Blocks:** 1-2 second block times vs 12 seconds on Ethereum
* **Enhanced Security:** Byzantine Fault Tolerant consensus prevents attacks

### v0.5.0 Compatibility Enhancements <a href="#v050-compatibility-enhancements" id="v050-compatibility-enhancements"></a>

#### EVM Equivalency Improvements <a href="#evm-equivalency-improvements" id="evm-equivalency-improvements"></a>

**EOA Code Delegation (EIP-7702):**

* **What:** Externally owned accounts can temporarily execute smart contract code
* **Benefit:** Account abstraction, batched operations, enhanced wallet functionality
* **Use Case:** Multi-sig wallets, automated strategies, custom transaction validation

**Historical Block Hash Access (EIP-2935):**

* **What:** BLOCKHASH opcode now provides reliable access to historical block hashes
* **Benefit:** Smart contracts can access up to 8192 previous block hashes (configurable)
* **Use Case:** Protocols requiring verifiable randomness or block-based logic

**Access List Optimization:**

* **What:** New `eth_createAccessList` RPC method for transaction optimization
* **Benefit:** Generate access lists to reduce transaction costs
* **Compatibility:** Matches Ethereum's access list functionality exactly

**Enhanced Transaction Responses:**

* **What:** Added `max_used_gas` field to transaction responses
* **Benefit:** Better gas estimation and transaction analytics
* **Use Case:** Tools can optimize gas usage more effectively

#### Performance Optimizations <a href="#performance-optimizations" id="performance-optimizations"></a>

**Smart Gas Estimation:**

* **Plain Transfer Detection:** Simple ETH transfers return 21000 gas immediately (\~90% faster)
* **Optimistic Bounds:** Complex transactions use initial execution results for better estimation bounds
* **Result:** Significantly faster `eth_estimateGas` for all transaction types

**Mempool Configurability:**

* **What:** Transaction pool limits, timeouts, and priority functions now fully configurable
* **Benefit:** Chains can optimize for their specific throughput and economic requirements
* **Example:** High-frequency DeFi chains can increase pool sizes, games can use custom priority

#### Transaction Pool <a href="#transaction-pool" id="transaction-pool"></a>

**Standard Ethereum mempool behavior is fully supported.** Transactions are ordered by gas price and nonce, multiple transactions per account per block work perfectly.

\*\*Optional Enhancement:\*\* An experimental two-tiered mempool is available that adds intelligent nonce gap handling and automatic transaction promotion. This is purely optional - the standard mempool works exactly like Ethereum.

### Migration from Ethermint/Evmos <a href="#migration-from-ethermintevmos" id="migration-from-ethermintevmos"></a>

Key breaking changes when migrating:

1. **Chain ID Format:** Cannot use Ethermint format (`evmos_9001-2`). Must configure two separate IDs.
2. **Module Names:** `x/evm` → `x/vm`
3. **Parameter Types:** Several parameter type changes (e.g., MinGasPrice: string → sdk.Dec)
4. **Precompile Addresses:** Different addresses than Ethermint

Detailed steps for migrating from Ethermint-based chains.

### Security & Reliability <a href="#security-andamp-reliability" id="security-andamp-reliability"></a>

**Battle-tested and audited:**

* **Security Audited:** Full audit by Sherlock (July 2025)
* **Production Ready:** Core EVM implementation is stable and deployed on multiple mainnets
* **Enhanced Security:** No reorganization attacks possible due to instant finality
* **Byzantine Fault Tolerant:** Requires 2/3+ validator stake for any state changes
* **Proven Stack:** Built on Ontomir SDK and 'CometBFT', powering hundreds of chains

### Resources <a href="#resources" id="resources"></a>

Add EVM support to your Ontomir chain Deep dive into architecture and design Source code and issue tracker Using Ontomir functionality through Solidity
