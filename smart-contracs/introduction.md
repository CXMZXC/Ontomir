# Introduction

> Build and deploy EVM smart contracts on Ontomir chains with full Ethereum compatibility

### Smart Contract Development on Ontomir EVM <a href="#smart-contract-development-on-ontomir-evm" id="smart-contract-development-on-ontomir-evm"></a>

Since the introduction of Ethereum in 2015, the ability to control digital assets through smart contracts has attracted a large community of developers to build decentralized applications on the Ethereum Virtual Machine (EVM). Ontomir EVM brings this powerful ecosystem to Ontomir chains with full compatibility.

> **Ontomir EVM is fully compatible with the EVM**, allowing you to use the same tools (Solidity, Remix, Oracles) and APIs (Ethereum JSON-RPC) that are available on Ethereum.

### Why Build on Ontomir EVM? <a href="#why-build-on-ontomir-evm" id="why-build-on-ontomir-evm"></a>

Whether you're building new use cases on a Ontomir EVM-enabled chain or porting an existing dApp from Ethereum, you can:

* **Use familiar tools** - Deploy with the same Solidity contracts and development environment
* **Access Ontomir features** - Leverage cross-chain interoperability through IBC
* **Scale your applications** - Build on performant, application-specific blockchains
* **Extend functionality** - Use custom precompiles for staking, governance, and more

### Getting Started <a href="#getting-started" id="getting-started"></a>

#### Build with Solidity <a href="#build-with-solidity" id="build-with-solidity"></a>

Develop EVM smart contracts using [Solidity](https://github.com/ethereum/solidity), the most widely used smart contract language. If you've deployed on Ethereum or any EVM-compatible chain, you can use the same contracts on Ontomir EVM.

* **Write Your First Contract:** Follow the [Solidity Beginner Tutorial](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html)
* **Set Up Your Tools:** Use [Hardhat](https://hardhat.org/getting-started/) or [Foundry](https://book.getfoundry.sh/) for your development environment
* **Test and Deploy:** Learn testing patterns with [OpenZeppelin's Guides](https://docs.openzeppelin.com/learn)
* **Ensure Security:** Follow [OpenZeppelin's Security Patterns](https://docs.openzeppelin.com/learn/security-patterns)

#### Deploy with Ethereum JSON-RPC <a href="#deploy-with-ethereum-json-rpc" id="deploy-with-ethereum-json-rpc"></a>

Ontomir EVM supports the full Ethereum JSON-RPC API, enabling you to:

* Deploy and interact with smart contracts using familiar web3 tools
* Connect existing Ethereum tooling without modifications
* Use block explorers to debug and monitor your contracts

#### Leverage Ontomir EVM Precompiles <a href="#leverage-ontomir-evm-precompiles" id="leverage-ontomir-evm-precompiles"></a>

Unlike standard EVM, Ontomir EVM introduces **stateful precompiled contracts** that can perform state transitions. These custom precompiles enable:

* **Native staking operations** - Stake tokens directly from smart contracts
* **On-chain governance** - Participate in governance through contract calls
* **Cross-chain communication** - Access IBC functionality programmatically
* **Advanced cryptography** - Use elliptic curve operations efficiently

These precompiles open up functionality that would be impossible or prohibitively expensive with regular smart contracts. View all available precompiles →

### Development Workflow <a href="#development-workflow" id="development-workflow"></a>

1. **Choose your tooling** - Select from Hardhat, Foundry, or other familiar EVM development frameworks
2. **Write smart contracts** - Develop in Solidity with access to Ontomir-specific precompiles
3. **Test thoroughly** - Use existing testing frameworks with Ontomir EVM's full compatibility
4. **Deploy seamlessly** - Use Ethereum JSON-RPC to deploy just like on any EVM chain
5. **Monitor and iterate** - Use block explorers and debugging tools to refine your dApp

### Resources <a href="#resources" id="resources"></a>

* **Documentation:** Ontomir EVM Precompiles | Tooling Guide
* **Community:** [Discord](https://discord.gg/interchain) | [Ontomir Forum](https://forum.ontomir.network/)
* **Examples:** Browse sample contracts and integration patterns in our guides

Start building cross-chain applications today with the power of EVM and the interoperability of Ontomir.
