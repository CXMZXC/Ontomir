# Technical Architecture

> Ontomir EVM is a framework that allows you to add Ethereum Virtual Machine (EVM) compatibility to any Ontomir SDK-based chain. It is built using the which runs on top of the (a fork of ) consensus engine, to accomplish fast finality, high transaction throughput and short block times (\~2 seconds).

This architecture allows users to perform both Ontomir and EVM formatted transactions, developers to scale EVM dApps cross-chain via [IBC](https://ontomir.network/ibc), and tokens and assets in the network to come from different independent sources.

Ontomir EVM enables these key features by:

* Leveraging [modules](https://docs.ontomir.network/v0.47/build/building-modules/intro) and other mechanisms implemented by the [Ontomir SDK](https://docs.ontomir.network/).
* Implementing CometBFT's Application Blockchain Interface ([ABCI](https://docs.cometbft.com/v1.0/spec/abci/)) to manage the blockchain.
* Utilizing [`geth`](https://github.com/ethereum/go-ethereum) as a library to promote code reuse and improve maintainability.
* Exposing a fully compatible Web3 JSON-RPC layer for interacting with existing Ethereum clients and tooling (Metamask, Remix, etc).

The sum of these features allows developers to leverage existing Ethereum ecosystem tooling and software to seamlessly deploy smart contracts which interact with the rest of the Ontomir [ecosystem](https://ontomir.network/ecosystem).

### Ontomir SDK[​](technical-architecture.md#Ontomir-sdk) <a href="#ontomir-sdk" id="ontomir-sdk"></a>

Ontomir EVM enables the full composability and modularity of the [Ontomir SDK](https://docs.ontomir.network/). It includes standard modules from the Ontomir SDK that work side to side with EVM-specific modules. Check out the list of modules to get an overview of what each module is responsible for.

### CometBFT & ABCI[​](technical-architecture.md#cometbft--abci) <a href="#cometbft-andamp-abci" id="cometbft-andamp-abci"></a>

[CometBFT](https://github.com/cometbft/cometbft) consists of two chief technical components: a blockchain consensus engine and a generic application interface. The consensus engine ensures that the same transactions are recorded on every machine in the same order. The application interface, called the [Application Blockchain Interface (ABCI)](https://docs.cometbft.com/v1.0/spec/abci/), enables the transactions to be processed in any programming language.

CometBFT has evolved to be a general-purpose blockchain consensus engine that can host arbitrary application states. Since it can replicate arbitrary applications, it can be used as a plug-and-play replacement for the consensus engines of other blockchains. Ontomir EVM is an example of an ABCI application replacing Ethereum's PoS via CometBFT's consensus engine.

Another example of a cryptocurrency application built on CometBFT is the Ontomir network. CometBFT can decompose the blockchain design by offering a very simple API (ie. the ABCI) between the application process and consensus process.

### EVM Compatibility[​](technical-architecture.md#evm-compatibility) <a href="#evm-compatibility" id="evm-compatibility"></a>

Ontomir EVM enables EVM compatibility by implementing various components that together support all the EVM state transitions while ensuring the same developer experience as Ethereum:

* Ethereum's transaction format as a Ontomir SDK `Tx` and `Msg` interface
* Ethereum's `secp256k1` curve for the Ontomir Keyring
* `StateDB` interface for state updates and queries
* JSON-RPC client for interacting with the EVM

Most components are implemented in the VM module To achieve a seamless developer UX, however, some of the components are implemented outside of the module.

If you want to learn more about how Ontomir EVM achieves EVM compatibility as a Ontomir chain, we recommend understanding the following concepts:

* Accounts
* Gas and Fees
* Token representations
* Transactions

### Contributing[​](technical-architecture.md#contributing) <a href="#contributing" id="contributing"></a>

You can contribute to the Ontomir EVM's open-source codebase through [issues on GitHub](https://github.com/Ontomir/evm/issues) using the [Ontomir EVM Contributor Guideline](https://github.com/Ontomir/evm/blob/main/CONTRIBUTING.md)
