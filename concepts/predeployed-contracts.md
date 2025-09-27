# Predeployed Contracts

> Understanding predeployed contracts - what they are, why they're used, and how they differ from precompiles

### What Are Predeployed Contracts? <a href="#what-are-predeployed-contracts" id="what-are-predeployed-contracts"></a>

Predeployed contracts (also called preinstalls) are regular EVM smart contracts with their bytecode deployed at predetermined addresses, either at chain genesis or through governance mechanisms. Unlike precompiles which are native Go implementations built into the chain binary, predeployed contracts:

* Contain actual EVM bytecode that can be executed by the EVM
* Consume gas like any regular smart contract
* Can be verified on block explorers
* Are immutable once deployed (no upgrades possible)

### Why Use Predeployed Contracts? <a href="#why-use-predeployed-contracts" id="why-use-predeployed-contracts"></a>

Predeployed contracts represent a fundamental pattern in EVM-based chains that provides several key advantages:

#### Standardization <a href="#standardization" id="standardization"></a>

By deploying essential contracts at known addresses, chains ensure compatibility with existing Ethereum tooling and libraries that expect these contracts at specific locations.

#### Developer Experience <a href="#developer-experience" id="developer-experience"></a>

Developers don't need to deploy their own versions of common utilities, saving deployment costs and reducing complexity.

#### Cross-chain Consistency <a href="#cross-chain-consistency" id="cross-chain-consistency"></a>

Using the same addresses across different chains enables easier multi-chain development and deployment strategies.

#### Ecosystem Alignment <a href="#ecosystem-alignment" id="ecosystem-alignment"></a>

Many tools, wallets, and dApps in the Ethereum ecosystem expect certain contracts (like Multicall3) at specific addresses.

### Comparison with Precompiles <a href="#comparison-with-precompiles" id="comparison-with-precompiles"></a>

Understanding the distinction between predeployed contracts and precompiles is crucial for choosing the right approach for your implementation:

| Aspect             | Predeployed Contracts         | Precompiles                        |
| ------------------ | ----------------------------- | ---------------------------------- |
| **Implementation** | EVM bytecode stored on-chain  | Native Go code in chain binary     |
| **Gas Costs**      | Standard EVM gas pricing      | Custom, typically lower gas costs  |
| **Deployment**     | Must be explicitly deployed   | Always available when enabled      |
| **Upgradeability** | Immutable once deployed       | Can be modified via chain upgrades |
| **Verification**   | Verifiable on block explorers | Source not visible on-chain        |
| **Address Range**  | Any valid address             | Special range (0x1-0x9FF)          |
| **Storage**        | Uses regular contract storage | Can access native Ontomir state    |

### When to Choose Predeployed Contracts <a href="#when-to-choose-predeployed-contracts" id="when-to-choose-predeployed-contracts"></a>

Use predeployed contracts when:

* You need standard Ethereum contracts at expected addresses
* Full EVM compatibility is required
* Contract source verification is important
* The functionality doesn't require native chain integration

Use precompiles when:

* You need optimized gas costs for frequently used operations
* Direct access to Ontomir SDK state is required
* The functionality requires native chain features
* Upgradeability through chain upgrades is needed

### Configuration <a href="#configuration" id="configuration"></a>

#### Genesis Configuration <a href="#genesis-configuration" id="genesis-configuration"></a>

Predeployed contracts are configured during chain genesis through the `preinstalls` parameter. This defines which contracts are deployed at specific addresses during network initialization.

```
{
  "app_state": {
    "evm": {
      "preinstalls": [
        {
          "address": "0x4e59b44847b379578588920ca78fbf26c0b4956c",
          "bytecode": "0x...",  
          "name": "CREATE2"
        },
        {
          "address": "0xcA11bde05977b3631167028862bE2a173976CA11",
          "bytecode": "0x...",
          "name": "Multicall3"  
        }
      ]
    }
  }
}
```

For complete EVM genesis configuration including precompiles and fee market parameters, see the \[Node Configuration]\(/docs/evm/next/documentation/getting-started/node-configuration#genesis-json) reference.

### Common Examples <a href="#common-examples" id="common-examples"></a>

The Ontomir EVM includes several default predeployed contracts:

* **CREATE2** (`0x4e59b44847b379578588920ca78fbf26c0b4956c`): Deterministic contract deployment
* **Multicall3** (`0xcA11bde05977b3631167028862bE2a173976CA11`): Batch multiple calls in one transaction
* **Permit2** (`0x000000000022D473030F116dDEE9F6B43aC78BA3`): Universal token approval system
* **Safe Factory** (`0x4e1DCf7AD4e460CfD30791CCC4F9c8a4f820ec67`): Deploy Safe multisig wallets
* Precompiles - Native chain implementations exposed as smart contracts
* Gas and Fees - Understanding gas consumption for predeployed contracts
* Chain ID - How chain identity affects contract addresses
* Implementation Guide - Technical details for deploying and managing preinstalled contracts
