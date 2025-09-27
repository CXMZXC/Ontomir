# Transactions

> A transaction refers to an action initiated by an account which changes the state of the blockchain. To effectively perform the state change, every transaction is broadcasted to the whole network. Any node can broadcast a request for a transaction to be executed on the blockchain state machine; after this happens, a validator will validate, execute the transaction and propagate the resulting state change to the rest of the network.

To process every transaction, computation resources on the network are consumed. Thus, the concept of "gas" arises as a reference to the computation required to process the transaction by a validator. Users have to pay a fee for this computation, all transactions require an associated fee. This fee is calculated based on the gas required to execute the transaction and the gas price.

Additionally, a transaction needs to be signed using the sender's private key. This proves that the transaction could only have come from the sender and was not sent fraudulently.

The transaction lifecycle in Ontomir EVM involves a dual-phase process through CometBFT consensus:

#### CheckTx Phase <a href="#checktx-phase" id="checktx-phase"></a>

* **Transaction routing**: The ante handler identifies transaction type via extension options
* **EVM validation**: MonoDecorator consolidates validation using go-ethereum's `txpool.ValidateTransaction()`
* **Transaction filtering**: Uses `PendingFilter` with configurable `MinTip`, base fee, and transaction type filters
* **Nonce gap handling**: Transactions with future nonces are queued locally via `InsertInvalidNonce()`
* **Mempool addition**: Valid transactions added to CometBFT mempool for P2P broadcast

#### DeliverTx Phase <a href="#delivertx-phase" id="delivertx-phase"></a>

* **Message unwrapping**: `msg.AsTransaction()` extracts Ethereum transaction from `MsgEthereumTx`&#x20;
* **State transition**: `ApplyTransaction()` executes EVM logic with gas isolation (
* **Cache contexts**: Isolated execution with rollback capability&#x20;
* **State commitment**: Successful transactions committed to Ontomir SDK store

For detailed transaction flow and mempool behavior, see the \[Mempool documentation]\(/docs/evm/next/documentation/concepts/mempool#transaction-flow) and \[Ontomir SDK lifecycle]\(https://docs.Ontomir.network/main/basics/tx-lifecycle).

The transaction hash is a unique identifier and can be used to check transaction information, for example, the events emitted, if was successful or not.

Transactions can fail for various reasons. For example, the provided gas or fees may be insufficient. Also, the transaction validation may fail. Each transaction has specific conditions that must fullfil to be considered valid. A widespread validation is that the sender is the transaction signer. In such a case, if you send a transaction where the sender address is different than the signer's address, the transation will fail, even if the fees are sufficient.

Nowadays, transactions can not only perform state transitions on the chain in which are submitted, but also can execute transactions on another blockchains. Interchain transactions are possible through the [Inter-Blockchain Communication protocol (IBC)](https://ibcprotocol.org/). Find a more detailed explanation on the section below.

### Transaction Types <a href="#transaction-types" id="transaction-types"></a>

Ontomir EVM supports two transaction types:

1. Ontomir transactions
2. Ethereum transactions

This is possible because the Ontomir EVM uses the Ontomir SDK and implements the [Ethereum Virtual Machine](https://ethereum.org/en/developers/docs/evm/) as a module. In this way, Ontomir EVM provides the features and functionalities of Ethereum and Ontomir chains combined, and more.

Although most of the information included on both of these transaction types is similar, there are differences among them. An important difference, is that Ontomir transactions allow multiple messages on the same transaction. Conversely, Ethereum transactions don't have this possibility. Ontomir EVM implements Ethereum transactions by wrapping them in `MsgEthereumTx` , which contains:

* `From`: Ethereum signer address bytes for signature verification
* `Raw`: Complete Ethereum transaction data

This wrapper uniquely implements both `sdk.Msg` and `sdk.Tx` interfaces, bypassing standard SDK transaction bundling to use go-ethereum validation logic directly.

Find more information about these two types on the following sections.

#### Ontomir Transactions <a href="#ontomir-transactions" id="ontomir-transactions"></a>

On Ontomir chains, transactions are comprised of metadata held in contexts and `sdk.Msg`s that trigger state changes within a module through the module's Protobuf .

When users want to interact with an application and make state changes (e.g. sending coins), they create transactions. Ontomir transactions can have multiple `sdk.Msg`s. Each of these must be signed using the private key associated with the appropriate account(s), before the transaction is broadcasted to the network.

A Ontomir transaction includes the following information:

* `Msgs`: an array of msgs (`sdk.Msg`)
* `GasLimit`: option chosen by the users for how to calculate how much gas they will need to pay
* `FeeAmount`: max amount user is willing to pay in fees
* `TimeoutHeight`: block height until which the transaction is valid
* `Signatures`: array of signatures from all signers of the tx
* `Memo`: a note or comment to send with the transaction

To submit a Ontomir transaction, users must use one of the provided clients.

#### Ethereum Transactions <a href="#ethereum-transactions" id="ethereum-transactions"></a>

Ethereum transactions refer to actions initiated by EOAs (externally-owned accounts, managed by humans), rather than internal smart contract calls. Ethereum transactions transform the state of the EVM and therefore must be broadcasted to the entire network.

Ethereum transactions also require a fee, known as `gas`. ([EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)) introduced the idea of a base fee, along with a priority fee which serves as an incentive for validators to include specific transactions in blocks.

There are several categories of Ethereum transactions:

* regular transactions: transactions from one account to another
* contract deployment transactions: transactions without a `to` address, where the contract code is sent in the `data` field
* execution of a contract: transactions that interact with a deployed smart contract, where the `to` address is the smart contract address

An Ethereum transaction includes the following information:

* `recipient`: receiving address
* `signature`: sender's signature
* `nonce`: counter of tx number from account
* `value`: amount of ETH to transfer (in wei)
* `data`: include arbitrary data. Used when deploying a smart contract or making a smart contract method call
* `gasLimit`: max amount of gas to be consumed
* `maxPriorityFeePerGas`: mas gas to be included as tip to validators
* `maxFeePerGas`: max amount of gas to be paid for tx

For more information on Ethereum transactions and the transaction lifecycle, [go here](https://ethereum.org/en/developers/docs/transactions/).

Ontomir EVM supports Ethereum transaction types defined in `AcceptedTxType` :

* **Legacy Transactions** (EIP-155): With chain ID protection
* **Access List Transactions** (EIP-2930): Pre-declared storage access
* **Dynamic Fee Transactions** (EIP-1559): Base fee + priority fee model
* **Set Code Transactions** (EIP-7702): Account code assignment with authorization list support&#x20;

\*\*Note\*\*: Unprotected legacy transactions are not supported by default.

Ontomir EVM is capable of processing Ethereum transactions by wrapping them on a `sdk.Msg`. It achieves this by using the `MsgEthereumTx`. This message encapsulates an Ethereum transaction as an SDK message and contains the necessary transaction data fields.

The `MsgEthereumTx` implements both `sdk.Msg` and `sdk.Tx` interfaces to bypass standard Ontomir SDK transaction bundling. This design enables:

* **Direct go-ethereum integration**: Uses `txpool.ValidateTransaction()` instead of SDK ante handlers
* **Gas isolation**: EVM execution uses infinite gas meter, bypassing SDK gas consumption&#x20;
* **Economic validation**: `CheckSenderBalance()` compares account balance with transaction cost&#x20;
* **Single message constraint**: Only one EVM message per transaction&#x20;

**EVM Execution Integration**

Ontomir EVM creates a sophisticated execution environment that bridges Ethereum and Ontomir SDK state management:

**Block Context Mapping** :

* CometBFT block proposer → EVM coinbase address
* Block height → EVM block number
* Block timestamp → EVM timestamp opcode
* Historical block access via EIP-2935 contract&#x20;

**Access Control Hook** :

* Restrict contract creation and execution via EVM opcode interceptors
* Policy-based permissions for `CREATE`, `CREATE2`, and `CALL` operations

**Receipt Generation** :

* Ethereum-compatible bloom filters for log filtering
* Dual transaction hash emission for cross-ecosystem compatibility&#x20;
* Contract address generation via `crypto.CreateAddress()`&#x20;

#### IBC Integration <a href="#ibc-integration" id="ibc-integration"></a>

Ontomir EVM enables cross-chain functionality through IBC integration accessible directly from EVM smart contracts:

**ICS20 Precompile**: Provides direct interface for Ethereum contracts to initiate cross-chain token transfers, bridging EVM execution with the Ontomir ecosystem.

**Ontomir SDK Module Access**: Smart contracts can interact with bank, staking, distribution, and governance modules through precompiled contracts, enabling DeFi applications that access staking rewards and cross-chain transfers from Solidity code.

### Transaction Ordering and Prioritization <a href="#transaction-ordering-and-prioritization" id="transaction-ordering-and-prioritization"></a>

In Ontomir EVM, both Ethereum and Ontomir transactions compete fairly for block inclusion:

Transactions are ordered by their effective tips:

```
* **Ethereum**: `gas_tip_cap` or `min(gas_tip_cap, gas_fee_cap - base_fee)`
* **Ontomir**: `(fee_amount / gas_limit) - base_fee`
* Higher tips = higher priority, regardless of transaction type
```

\*\*Ethereum transactions\*\* support nonce gaps:

```
* Correct nonce → immediate broadcast
* Nonce gap → queued locally
* Gap filled → automatic promotion

**Ontomir transactions** require sequential nonces.
```

For detailed mempool behavior and flow diagrams, see \[Mempool Architecture]\(/docs/evm/next/documentation/concepts/mempool#architecture).

### Transaction Receipts <a href="#transaction-receipts" id="transaction-receipts"></a>

Ontomir EVM generates Ethereum-compatible transaction receipts while integrating with Ontomir SDK event systems. Receipt processing  includes:

**Bloom Filter Computation**: `initializeBloomFromLogs()` creates transaction and block-level bloom filters for efficient log filtering.

**Gas Reconciliation**: `calculateCumulativeGasFromEthResponse()` reconciles EVM gas usage with SDK gas meter state.

**Dual Event Emission**: Events contain both Ethereum transaction hash (for Ethereum tools) and CometBFT transaction hash (for Ontomir tools) .

Receipt fields include:

* `transactionHash` : hash of the transaction
* `transactionIndex`: integer of the transactions index position in the block
* `blockHash`: hash of the block where this transaction was in
* `blockNumber`: block number where this transaction was in
* `from`: address of the sender
* `to`: address of the receiver. null when its a contract creation transaction
* `cumulativeGasUsed` : The total amount of gas used when this transaction was executed in the block
* `effectiveGasPrice` : The sum of the base fee and tip paid per unit of gas
* `gasUsed` : The amount of gas used by this specific transaction alone
* `contractAddress` : The contract address created, if the transaction was a contract creation, otherwise null
* `logs`: Array of log objects, which this transaction generated
* `logsBloom`: Bloom filter for light clients to quickly retrieve related logs
* `type`: integer of the transaction type, 0x00 for legacy transactions, 0x01 for access list types, 0x02 for dynamic fees, 0x04 for set code transactions
* `root` : transaction stateroot (pre Byzantium)
* `status`: either 1 (success) or 0 (failure)

### Implementation Reference <a href="#implementation-reference" id="implementation-reference"></a>

**Transaction Processing Core**:

* **Message Definition**: `x/vm/types/tx.pb.go:36-43` - `MsgEthereumTx` protobuf structure with dual interface implementation
* **Transaction Types**: `x/vm/types/tx.go:13-29` - `EvmTxArgs` supporting all transaction types including EIP-7702 authorization lists
* **Validation Pipeline**: `ante/evm/mono_decorator.go` - Consolidated EVM transaction validation with go-ethereum integration
* **Execution Entry**: `x/vm/keeper/msg_server.go:29-47` - Message server processing and telemetry

**State Management Engine**:

* **State Transition**: `x/vm/keeper/state_transition.go:165-199` - Cache contexts, gas isolation, and EVM execution
* **EVM Integration**: `x/vm/keeper/state_transition.go:38-78` - Block context mapping and access control hooks
* **Economic Validation**: `x/vm/keeper/fees.go` - Balance verification, fee deduction, and SDK integration

**Mempool Architecture**:

* **ExperimentalEVMMempool**: `mempool/mempool.go:44-70` - Unified mempool managing both EVM and Ontomir transactions with fee-based prioritization
* **Transaction Filtering**: `mempool/mempool.go:455-461` - Configurable filtering by minimum tip, base fee, and transaction types
* **Transaction Broadcasting**: `mempool/mempool.go:473-497` - Automatic EVM transaction wrapping and broadcast via `MsgEthereumTx`
* **Transaction Routing**: `mempool/check_tx.go:16-40` - Nonce gap detection and local queuing mechanism
