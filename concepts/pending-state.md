# Pending State

> When a transaction is submitted to the Ethereum network, it first goes into the pending status, waiting to be executed by the nodes. A transaction can be in the pending state for a longer duration if the gas price is set very low in the transaction and the nodes are busy processing other higher gas price transactions.

During the pending state, the transaction initiator is allowed to change the transaction fields at any time. They can do so by sending another transaction with the same nonce.

### Prerequisite Readings[​](pending-state.md#prerequisite-readings) <a href="#prerequisite-readings" id="prerequisite-readings"></a>

* Mempool Architecture

### Ontomir EVM vs Ethereum[​](pending-state.md#Ontomir-evm-vs-ethereum) <a href="#ontomir-evm-vs-ethereum" id="ontomir-evm-vs-ethereum"></a>

In Ethereum, pending blocks are generated as they are queued for production by miners. These pending blocks include pending transactions that are picked out by miners, based on the highest reward paid in gas. This mechanism exists as block finality is not possible on the Ethereum network. Blocks are committed with probabilistic finality, which means that transactions and blocks become less likely to become reverted as more time (and blocks) passes.

Ontomir EVM is designed differently as it uses [CometBFT](https://docs.cometbft.com/v1.0/) consensus which provides instant finality for transactions. While there is no concept of "pending blocks" that can be reorganized, Ontomir EVM implements an **experimental EVM mempool** that provides Ethereum-compatible pending state functionality.

### EVM-Compliant Mempool <a href="#evm-compliant-mempool" id="evm-compliant-mempool"></a>

The experimental EVM mempool introduces a two-tiered system that brings Ethereum-like pending state behavior to Ontomir EVM:

#### Transaction States <a href="#transaction-states" id="transaction-states"></a>

Transactions with correct nonces that are immediately executable. These are in the public mempool, broadcast to peers, and ready for block inclusion. Transactions with future nonces (nonce gaps) that are stored locally. These wait for earlier transactions to execute before being promoted to pending.

#### Key Differences from Traditional Ontomir <a href="#key-differences-from-traditional-ontomir" id="key-differences-from-traditional-ontomir"></a>

1. **Nonce Gap Handling**: Unlike traditional Ontomir chains that reject out-of-order transactions, the EVM mempool queues them locally until gaps are filled.
2. **Fee-Based Priority**: Both EVM and Ontomir transactions compete fairly based on their effective tips rather than FIFO ordering:
   * **EVM transactions**: Priority = `gas_tip_cap` or `min(gas_tip_cap, gas_fee_cap - base_fee)`
   * **Ontomir transactions**: Priority = `(fee_amount / gas_limit) - base_fee`
3. **Transaction Replacement**: Supports replacing pending transactions with higher fee versions using the same nonce, enabling "speed up" functionality common in Ethereum wallets.

### Pending State Queries[​](pending-state.md#pending-state-queries) <a href="#pending-state-queries" id="pending-state-queries"></a>

With the experimental EVM mempool, pending state queries now reflect a more Ethereum-compatible view:

#### Transaction Pool Inspection <a href="#transaction-pool-inspection" id="transaction-pool-inspection"></a>

The mempool provides dedicated RPC methods to inspect pending and queued transactions:

Returns counts of pending and queued transactions:

````
```json
{
  "pending": "0x10",  // 16 pending transactions
  "queued": "0x5"     // 5 queued transactions
}
```
````

Returns detailed information about all transactions in the pool, organized by account and nonce. Provides human-readable summaries of transactions for debugging.

See the [JSON-RPC Methods](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool-methods) documentation for complete details.

#### Pending State Behavior <a href="#pending-state-behavior" id="pending-state-behavior"></a>

When making queries with "pending" as the block parameter:

1. **Balance Queries**: Reflect the account balance after all pending transactions from that account are applied
2. **Nonce Queries**: Return the next available nonce considering all pending transactions
3. **Gas Estimates**: Account for pending transactions that may affect gas costs

Pending state queries are subjective to each node's local mempool view. Different nodes may return different results based on their transaction pool contents.

### JSON-RPC Calls Supporting Pending State[​](pending-state.md#json-rpc-calls-on-pending-transactions) <a href="#json-rpc-calls-supporting-pending-state" id="json-rpc-calls-supporting-pending-state"></a>

The following RPC methods support the `"pending"` block parameter:

* [`eth_getBalance`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_getbalance) - Get balance considering pending transactions
* [`eth_getTransactionCount`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_gettransactioncount) - Get next nonce considering pending transactions
* [`eth_getBlockTransactionCountByNumber`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_getblocktransactioncountbynumber) - Count pending transactions
* [`eth_getBlockByNumber`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_getblockbynumber) - Get pending block information
* [`eth_getTransactionByHash`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_gettransactionbyhash) - Retrieve pending transactions
* [`eth_getTransactionByBlockNumberAndIndex`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_gettransactionbyblockhashandindex) - Access specific pending transactions
* [`eth_sendTransaction`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth_sendtransaction) - Submit new transactions

Additionally, the `txpool_*` namespace provides specialized methods for mempool inspection:

* [`txpool_status`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool_status) - Get pool statistics
* [`txpool_content`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool_content) - View all pool transactions
* [`txpool_contentFrom`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool_contentfrom) - Filter by address
* [`txpool_inspect`](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool_inspect) - Human-readable summaries

### Practical Examples <a href="#practical-examples" id="practical-examples"></a>

#### Monitoring Transaction Status <a href="#monitoring-transaction-status" id="monitoring-transaction-status"></a>

```
// Check if transaction is pending
const tx = await provider.getTransaction(txHash);
if (tx && !tx.blockNumber) {
  console.log("Transaction is pending");

  // Check pool status
  const poolStatus = await provider.send("txpool_status", []);
  console.log(`Pool has ${poolStatus.pending} pending, ${poolStatus.queued} queued`);
}
```

#### Handling Nonce Gaps <a href="#handling-nonce-gaps" id="handling-nonce-gaps"></a>

```
// Send transactions with nonce gaps (they'll be queued)
await wallet.sendTransaction({nonce: 100, ...});  // Executes immediately
await wallet.sendTransaction({nonce: 102, ...});  // Queued (gap at 101)
await wallet.sendTransaction({nonce: 101, ...});  // Fills gap, both execute
```

#### Transaction Replacement <a href="#transaction-replacement" id="transaction-replacement"></a>

```
// Speed up a pending transaction
const originalTx = await wallet.sendTransaction({
  nonce: 100,
  gasPrice: parseUnits("20", "gwei")
});

// Replace with higher fee
const fasterTx = await wallet.sendTransaction({
  nonce: 100,  // Same nonce
  gasPrice: parseUnits("30", "gwei")  // Higher fee
});
```

### Architecture Details <a href="#architecture-details" id="architecture-details"></a>

For a detailed understanding of how the pending state is managed:

* See [Mempool Architecture](https://about/docs/evm/next/documentation/concepts/mempool#architecture) for the two-tier system design
* Review [Transaction Flow](https://about/docs/evm/next/documentation/concepts/mempool#transaction-flow) for state transitions
* Check Integration Guide for implementation details
