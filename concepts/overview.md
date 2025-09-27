# Overview

> Understanding how the EVM module provides Ethereum compatibility within the Ontomir SDK platform

The Ontomir EVM module implements a complete Ethereum Virtual Machine as a Ontomir SDK module, providing an Ethereum-compatible interface to the underlying Ontomir consensus and infrastructure while enabling access to the broader Ontomir ecosystem.

### Platform Integration <a href="#platform-integration" id="platform-integration"></a>

The EVM module sits atop the Ontomir SDK platform, leveraging its modular architecture:

* **Consensus Layer**: CometBFT provides Byzantine fault-tolerant consensus with instant finality
* **State Management**: Ontomir SDK's IAVL tree and KVStore handle state persistence
* **Account System**: Unified account model supporting both Ethereum and Ontomir address formats
* **Module Ecosystem**: Direct access to staking, governance, bank, and IBC modules through precompiles

### Ethereum Compatibility <a href="#ethereum-compatibility" id="ethereum-compatibility"></a>

The EVM module provides complete Ethereum compatibility, enabling all standard Ethereum tooling and workflows:

* **Smart Contracts**: Full EVM bytecode execution with identical gas costs and opcode behavior
* **Transaction Types**: Support for all Ethereum transaction formats including EIP-1559 and EIP-7702
* **JSON-RPC API**: Complete Ethereum RPC implementation for seamless tool integration
* **Development Tools**: Works with MetaMask, Hardhat, Foundry, Remix, and all Ethereum development frameworks

### Enhanced Features <a href="#enhanced-features" id="enhanced-features"></a>

#### Ontomir SDK Benefits <a href="#ontomir-sdk-benefits" id="ontomir-sdk-benefits"></a>

* **Instant Finality**: Transactions are final after one block (\~2 seconds) with no reorganizations
* **Cross-Chain Integration**: Native IBC support for interacting with other Ontomir chains
* **Modular Access**: Smart contracts can directly interact with staking, governance, and other Ontomir modules
* **Enhanced Security**: Byzantine fault-tolerant consensus with stake-based validator selection

### Key Differences from Standard Ethereum <a href="#key-differences-from-standard-ethereum" id="key-differences-from-standard-ethereum"></a>

* **Consensus**: CometBFT instead of proof-of-stake, providing instant finality
* **State Storage**: IAVL tree and Ontomir SDK KVStore instead of Merkle Patricia Tree
* **Fee Distribution**: Base fees distributed to validators instead of burned
* **Cross-Chain**: Native IBC integration for seamless interchain operations
* **Module Access**: Smart contracts can directly call Ontomir SDK modules

### Developer Experience <a href="#developer-experience" id="developer-experience"></a>

Developers can build on Ontomir EVM using familiar Ethereum tools and patterns:

* **Standard Tooling**: MetaMask, Hardhat, Foundry, Remix work without modification
* **Ethereum Libraries**: Web3.js, Ethers.js, and other libraries work seamlessly
* **Smart Contracts**: Deploy existing Ethereum contracts without changes
* **Enhanced Capabilities**: Access Ontomir modules and IBC through precompiled contracts

For detailed technical implementation, see individual concept pages for transactions, accounts, gas and fees, and precompiles.
