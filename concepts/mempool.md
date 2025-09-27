# Mempool

> Design and Rationale

### Overview <a href="#overview" id="overview"></a>

The EVM mempool is responsible for managing both EVM and Ontomir transactions in a unified pool, enabling Ethereum-compatible transaction flows including out-of-order transactions and nonce gap handling. It serves as a replacement for the default CometBFT FIFO mempool to support Ethereum tooling expectations while maintaining Ontomir SDK compatibility.

### Purpose and Design <a href="#purpose-and-design" id="purpose-and-design"></a>

The EVM mempool serves as a bridge between Ethereum's transaction management model and Ontomir SDK's consensus layer, enabling Ethereum-compatible transaction flows while maintaining the security and finality guarantees of CometBFT consensus.

#### Design Goals <a href="#design-goals" id="design-goals"></a>

**Ethereum Compatibility**: Enable Ethereum tooling to work without modification by supporting:

* Out-of-order transaction submission
* Nonce gap handling and automatic promotion
* Transaction replacement with higher fees
* Standard txpool RPC methods

**Ontomir Integration**: Maintain compatibility with Ontomir SDK patterns:

* Unified mempool for both EVM and Ontomir transactions
* Fee-based transaction prioritization
* Integration with existing ante handlers
* Preservation of consensus finality

#### Use Cases <a href="#use-cases" id="use-cases"></a>

**Complex Contract Deployments**: DeFi protocols like Uniswap deploy multiple interdependent contracts rapidly. The mempool handles these deployment scripts without modification by queuing transactions with nonce gaps until they can be executed in order.

**Batch Transaction Workflows**: Development tools and scripts often submit multiple transactions simultaneously, expecting the network to handle ordering and dependencies automatically.

**Transaction Replacement**: Users can speed up pending transactions by submitting replacement transactions with the same nonce but higher fees, following standard Ethereum patterns.

### Transaction Flow <a href="#transaction-flow" id="transaction-flow"></a>

#### 1. Transaction Submission <a href="#id-1-transaction-submission" id="id-1-transaction-submission"></a>

Users or other nodes submit transactions to the chain via JSON-RPC or P2P.

#### 2. CometBFT Reception <a href="#id-2-cometbft-reception" id="id-2-cometbft-reception"></a>

CometBFT receives the transactions and validates them in the app using CheckTx.

#### 3. CheckTx Routing <a href="#id-3-checktx-routing" id="id-3-checktx-routing"></a>

The CheckTx handler processes transactions with special handling for nonce gaps ([source](https://github.com/Ontomir/evm/blob/v0.4.1/mempool/check_tx.go)):

**Success Path** - Valid transactions with correct nonces pass through to the Comet mempool for broadcast.

**Nonce Gap Detection** - Transactions with future nonces are intercepted and queued locally:

```
// From mempool/check_tx.go
if err != nil {
    // detect if there is a nonce gap error (only returned for EVM transactions)
    if errors.Is(err, ErrNonceGap) {
        // send it to the mempool for further triage
        err := mempool.InsertInvalidNonce(request.Tx)
        if err != nil {
            return sdkerrors.ResponseCheckTxWithEvents(err, gInfo.GasWanted, gInfo.GasUsed, anteEvents, false), nil
        }
    }
    // anything else, return regular error
    return sdkerrors.ResponseCheckTxWithEvents(err, gInfo.GasWanted, gInfo.GasUsed, anteEvents, false), nil
}
```

**Other Failures** - Rejected and return error to client:

* **Insufficient fees**: Transactions with `GasFeeCap < BaseFee` fail with `ErrInsufficientFee`
* **Insufficient balance**: Transactions exceeding account balance
* **Invalid signature**: Malformed or invalid transaction signatures

Note: Only nonce gaps trigger local queuing. Fee-related failures result in immediate rejection.

#### 4. Comet Mempool Addition <a href="#id-4-comet-mempool-addition" id="id-4-comet-mempool-addition"></a>

Successfully validated transactions are added to the Comet mempool (FIFO).

#### 5. P2P Broadcast <a href="#id-5-p2p-broadcast" id="id-5-p2p-broadcast"></a>

Transactions in the Comet mempool are broadcast to other peers across the network.

#### 6. Block Building <a href="#id-6-block-building" id="id-6-block-building"></a>

When a validator is selected to propose a block, ProcessProposal uses the mempool to build blocks:

* Sorts transactions by account (fee priority) and nonce
* Pulls from both local queue and public pool
* Replaces lower-fee duplicates with higher-fee versions

#### 7. Automatic Promotion <a href="#id-7-automatic-promotion" id="id-7-automatic-promotion"></a>

The node periodically scans the local queue and promotes transactions when:

* Nonce gaps are filled (either in mempool or from on-chain state)
* Promoted transactions are re-broadcast to the network

### Transaction Lifecycle <a href="#transaction-lifecycle" id="transaction-lifecycle"></a>

Client submits tx

Validation

Valid & Executable

Nonce Gap Only

Invalid (fees, balance, etc.)

Gap filled

Re-broadcast

P2P propagation

Selected for block

Direct inclusion

In block

Submitted

CheckTx

Pending

Queued

Rejected

Promoted

Broadcast

BlockBuilding

Committed

### Architecture <a href="#architecture" id="architecture"></a>

![Mempool Architecture](https://mintcdn.com/Ontomir-docs/VsPAy4At7iyU59UI/assets/public/mempool_architecture.png?fit=max\&auto=format\&n=VsPAy4At7iyU59UI\&q=85\&s=0094769290820a7c0e8830338d4fad32)

The mempool uses a two-tiered system with local and public transaction pools.

#### Problem Statement <a href="#problem-statement" id="problem-statement"></a>

CometBFT rejects transactions with:

* Nonce gaps (non-sequential nonces)
* Out-of-order batches (common in deployment scripts)

Ethereum tooling expects these transactions to queue rather than fail.

#### Solution Architecture <a href="#solution-architecture" id="solution-architecture"></a>

To improve DevEx, a tiered approach was implemented: a local transaction pool handles queuing nonce-gapped transactions, upgrading transactions to CometBFT mempool which allows them to be gossipped to network peers and be included in blocks. This helps reduce network spam/DOS exposure while also enabling proper EVM transaction semantics.

The two-tiered approach:

* **Local queue**: Stores gapped transactions without network propagation, preventing invalid transaction gossip
* **Public mempool**: Contains only valid transactions, maintaining consensus integrity
* **Automatic promotion**: Moves transactions from local to public when gaps fill, ensuring inclusion once conditions are met

![Transaction Flow](https://mintcdn.com/Ontomir-docs/VsPAy4At7iyU59UI/assets/public/mempool_transaction_flow.png?fit=max\&auto=format\&n=VsPAy4At7iyU59UI\&q=85\&s=9f2f5dc794813f6ec8cc6724d84604d3)

#### Core Components <a href="#core-components" id="core-components"></a>

**CheckTx Handler** Intercepts nonce gap errors during validation, routes gapped transactions to the local queue, and returns success to maintain compatibility with Ethereum tooling that expects queuing behavior. Only nonce gaps are intercepted - other validation failures (insufficient fees, balance, etc.) are rejected immediately.

**TxPool** Direct port of Ethereum's transaction pool that manages both pending (executable) and queued (future) transactions. Handles promotion, eviction, and replacement according to Ethereum rules.

**LegacyPool** Stores non-executable transactions with nonce gaps, tracks dependencies between transactions, and automatically promotes them when gaps are filled. The queue contains only transactions waiting for earlier nonces - not transactions with insufficient fees.

**ExperimentalEVMMempool** Unified structure that manages both EVM and Ontomir transaction pools while providing a single interface for transaction insertion, selection, and removal.

#### Transaction States <a href="#transaction-states" id="transaction-states"></a>

**Queued (Local Storage)**:

* **Nonce gaps**: Transactions with nonce > expected nonce
* These are stored locally and promoted when gaps fill

**Rejected (Immediate Failure)**:

* **Insufficient fees**: `GasFeeCap < BaseFee`
* **Insufficient balance**: Transaction cost exceeds account balance
* **Invalid signature**: Malformed or improperly signed transactions
* **Gas limit exceeded**: Transactions exceeding block gas limit

Only nonce-gapped transactions are intercepted and queued. All other validation failures result in immediate rejection with error returned to the client. This combined approach preserves CometBFT's leading consensus mechanism while providing Ethereum's expected tx queuing behavior.

### API Reference <a href="#api-reference" id="api-reference"></a>

The mempool exposes Ethereum-compatible RPC methods for querying transaction pool state. See the [JSON-RPC Methods documentation](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool-methods) for detailed API reference:

* **`txpool_status`**: Get pending and queued transaction counts
* **`txpool_content`**: View all transactions in the pool
* **`txpool_contentFrom`**: View transactions from specific addresses
* **`txpool_inspect`**: Get human-readable transaction summaries

You can also explore these methods interactively using the RPC Explorer.

### Integration <a href="#integration" id="integration"></a>

For chain developers looking to integrate the mempool into their Ontomir SDK chain, see the \[EVM Mempool Integration Guide]\(/docs/evm/next/documentation/integration/mempool-integration) for complete setup instructions.

### Configuration <a href="#configuration" id="configuration"></a>

#### v0.5.0 Configuration Changes <a href="#v050-configuration-changes" id="v050-configuration-changes"></a>

**Breaking Change:** v0.5.0 replaces pre-built pool objects with configuration-based instantiation for better flexibility.

**What Changed**

**Before v0.5.0 (Pre-built Objects):**

```
// Required pre-built pools
customTxPool := CreateCustomTxPool(...)
customOntomirPool := CreateCustomOntomirPool(...)

mempoolConfig := &evmmempool.EVMMempoolConfig{
    TxPool:     customTxPool,      // ← REMOVED in v0.5.0
    OntomirPool: customOntomirPool,  // ← REMOVED in v0.5.0
    AnteHandler: app.GetAnteHandler(),
}
```

**After v0.5.0 (Configuration Objects):**

```
// Configuration-based approach
mempoolConfig := &evmmempool.EVMMempoolConfig{
    LegacyPoolConfig: &legacypool.Config{...}, // ← NEW in v0.5.0
    OntomirPoolConfig: &sdkmempool.PriorityNonceMempoolConfig{...}, // ← NEW in v0.5.0
    AnteHandler:   app.GetAnteHandler(),
    BroadcastTxFn: customBroadcastFunc, // ← NEW: Optional custom broadcast
    BlockGasLimit: 100_000_000,         // ← NEW: Configurable gas limit
    MinTip:        uint256.NewInt(1000000000), // ← NEW: Minimum tip filter (optional)
}
```

**Where to Configure**

**Application Code (`app/app.go`):**

* **Mempool instantiation** during app construction
* **Custom pool configurations** for chain-specific requirements
* **Broadcast function** customization (optional)

**No Configuration File Changes:**

* Mempool settings are **code-level configuration only**
* No `app.toml` or `config.toml` changes required
* Configuration happens during application startup

#### Mempool Configuration Options <a href="#mempool-configuration-options" id="mempool-configuration-options"></a>

v0.5.0 introduces comprehensive configuration options for customizing mempool behavior, replacing previously hard-coded parameters with flexible configuration objects.

**Basic Configuration**

```
// Minimal configuration with defaults
mempoolConfig := &evmmempool.EVMMempoolConfig{
    AnteHandler:   app.GetAnteHandler(),
    BlockGasLimit: 100_000_000, // or 0 for auto-default
}

evmMempool := evmmempool.NewExperimentalEVMMempool(
    app.CreateQueryContext,
    logger,
    app.EVMKeeper,
    app.FeeMarketKeeper,
    app.txConfig,
    app.clientCtx,
    mempoolConfig,
)
```

**Advanced Configuration**

```
// Custom configuration for high-throughput chains
mempoolConfig := &evmmempool.EVMMempoolConfig{
    LegacyPoolConfig: &legacypool.Config{
        AccountSlots: 32,           // Transactions per account (default: 16)
        GlobalSlots:  8192,         // Total pending transactions (default: 5120)  
        AccountQueue: 128,          // Queued per account (default: 64)
        GlobalQueue:  2048,         // Total queued transactions (default: 1024)
        Lifetime:     6*time.Hour,  // Transaction lifetime (default: 3h)
        PriceLimit:   2,            // Min gas price in wei (default: 1)
        PriceBump:    15,           // Replacement bump % (default: 10)
        // Source: mempool/txpool/legacypool/legacypool.go:168-178
        Journal:      "txpool.rlp", // Persistence file (default: "transactions.rlp")
        Rejournal:    2*time.Hour,  // Journal refresh interval (default: 1h)
    },
    OntomirPoolConfig: &sdkmempool.PriorityNonceMempoolConfig[math.Int]{
        TxPriority: sdkmempool.TxPriority[math.Int]{
            GetTxPriority: customPriorityFunction,
            Compare:       math.IntComparator,
            MinValue:      math.ZeroInt(),
        },
    },
    AnteHandler:   app.GetAnteHandler(),
    BroadcastTxFn: customBroadcastFunction, // Optional custom broadcast logic
    BlockGasLimit: 200_000_000,             // Custom block gas limit
}
```

**Custom Priority Functions**

```
// Example: Prioritize governance and staking transactions
func customPriorityFunction(goCtx context.Context, tx sdk.Tx) math.Int {
    // High priority for governance proposals
    for _, msg := range tx.GetMsgs() {
        if _, ok := msg.(*govtypes.MsgSubmitProposal); ok {
            return math.NewInt(1000000)
        }
        if _, ok := msg.(*stakingtypes.MsgDelegate); ok {
            return math.NewInt(100000)
        }
    }

    // Standard fee-based priority for other transactions
    feeTx, ok := tx.(sdk.FeeTx)
    if !ok {
        return math.ZeroInt()
    }

    fee := feeTx.GetFee().AmountOf("test")
    gas := feeTx.GetGas()
    if gas == 0 {
        return fee
    }
    return fee.QuoUint64(gas)
}
```

**Custom Broadcast Functions**

```
// Example: Rate-limited broadcasting
func customBroadcastFunction(txs []*ethtypes.Transaction) error {
    for i, tx := range txs {
        // Rate limit: max 10 tx/second
        if i > 0 && i%10 == 0 {
            time.Sleep(1 * time.Second)
        }

        // Custom broadcast logic
        if err := broadcastTransaction(tx); err != nil {
            return fmt.Errorf("failed to broadcast tx %s: %w", tx.Hash(), err)
        }
    }
    return nil
}
```

#### Configuration Parameter Details <a href="#configuration-parameter-details" id="configuration-parameter-details"></a>

**LegacyPoolConfig Parameters**

**Source:** [`mempool/txpool/legacypool/legacypool.go:168-178`](https://github.com/Ontomir/evm/blob/main/mempool/txpool/legacypool/legacypool.go#L168-178)

```
type Config struct {
    // Transaction pool capacity
    AccountSlots uint64 // Executable transactions per account (default: 16)
    GlobalSlots  uint64 // Total executable transactions (default: 5120)  
    AccountQueue uint64 // Non-executable transactions per account (default: 64)
    GlobalQueue  uint64 // Total non-executable transactions (default: 1024)

    // Economic parameters
    PriceLimit uint64 // Minimum gas price in wei (default: 1)
    PriceBump  uint64 // Minimum price bump % for replacement (default: 10)

    // Lifecycle management
    Lifetime  time.Duration // Transaction retention time (default: 3h)
    Journal   string        // Persistence file (default: "transactions.rlp")
    Rejournal time.Duration // Journal refresh interval (default: 1h)

    // Local transaction handling
    Locals   []common.Address // Addresses treated as local (no gas limit)
    NoLocals bool            // Disable local transaction handling
}
```

**OntomirPoolConfig Parameters**

**Source:** Ontomir SDK mempool types

```
type PriorityNonceMempoolConfig[C comparable] struct {
    TxPriority    TxPriority[C]    // Custom priority calculation function
    OnRead        func(tx sdk.Tx)  // Callback when transaction is read
    TxReplacement TxReplacement[C] // Transaction replacement rules
}

// Priority calculation interface
type TxPriority[C comparable] struct {
    GetTxPriority func(context.Context, sdk.Tx) C // Calculate transaction priority
    Compare       func(a, b C) int                // Compare two priorities  
    MinValue      C                               // Minimum priority value
}
```

#### Migration from v0.4.x <a href="#migration-from-v04x" id="migration-from-v04x"></a>

**Step 1: Remove Pre-built Pools**

**Remove this code:**

```
// Delete these lines from your app.go
customTxPool := legacypool.New(config, blockchain)
customOntomirPool := sdkmempool.NewPriorityMempool(priorityConfig)

mempoolConfig := &evmmempool.EVMMempoolConfig{
    TxPool:     customTxPool,     // ← Remove
    OntomirPool: customOntomirPool, // ← Remove
}
```

**Step 2: Add Configuration Objects**

**Add this code:**

```
// Replace with configuration-based approach
mempoolConfig := &evmmempool.EVMMempoolConfig{
    // Convert custom pool settings to config
    LegacyPoolConfig: &legacypool.Config{
        AccountSlots: 32,    // Previously customTxPool.AccountSlots
        GlobalSlots:  8192,  // Previously customTxPool.GlobalSlots
        PriceLimit:   2,     // Previously customTxPool.PriceLimit
        // ... other settings from your old pool
    },
    OntomirPoolConfig: &sdkmempool.PriorityNonceMempoolConfig[math.Int]{
        TxPriority: priorityConfig.TxPriority, // Reuse existing priority logic
    },
    AnteHandler:   app.GetAnteHandler(),
    BlockGasLimit: 100_000_000, // NEW: Must specify
}
```

**Step 3: Update Imports**

**Add required imports:**

```
import (
    "time"
    "Ontomirsdk.io/math"
    "github.com/Ontomir/evm/mempool/txpool/legacypool"
    sdkmempool "github.com/Ontomir/Ontomir-sdk/types/mempool"
)
```

**Step 4: Handle New Required Parameters**

**New Required Parameter:**

```
mempoolConfig := &evmmempool.EVMMempoolConfig{
    BlockGasLimit: 100_000_000, // ← REQUIRED in v0.5.0 (was optional)
    AnteHandler:   app.GetAnteHandler(), // ← Still required
}
```

**Parameter Defaults:**

* If `BlockGasLimit` is set to `0`, defaults to `100_000_000`
* If `LegacyPoolConfig` is `nil`, uses `legacypool.DefaultConfig`
* If `OntomirPoolConfig` is `nil`, uses default priority mempool
* If `MinTip` is `nil`, no minimum tip enforcement
* `BroadcastTxFn` is optional (uses default broadcast if nil)

**Step 5: Initialization Location**

**Where to Add Configuration:**

```
// In app/app.go, during NewApp() function
func NewApp(...) *App {
    // ... other initialization

    // Configure mempool AFTER ante handler setup
    if evmtypes.GetChainConfig() != nil {
        mempoolConfig := &evmmempool.EVMMempoolConfig{
            AnteHandler:   app.GetAnteHandler(), // Must be set first
            BlockGasLimit: 100_000_000,
            MinTip:        uint256.NewInt(1000000000), // 1 gwei minimum tip
            // Add custom configs here
        }

        app.EVMMempool = evmmempool.NewExperimentalEVMMempool(
            app.CreateQueryContext,
            logger,
            app.EVMKeeper,
            app.FeeMarketKeeper,
            app.txConfig,
            app.clientCtx,
            mempoolConfig,
        )

        // Set as app mempool
        app.SetMempool(app.EVMMempool)
    }

    return app
}
```

#### Configuration Use Cases <a href="#configuration-use-cases" id="configuration-use-cases"></a>

**High-Throughput Chains**

```
// Optimize for maximum transaction volume
LegacyPoolConfig: &legacypool.Config{
    AccountSlots: 64,    // More transactions per account
    GlobalSlots:  16384, // Higher global capacity
    AccountQueue: 256,   // Larger queues
    GlobalQueue:  4096,
    PriceLimit:   1,     // Accept lower gas prices
}
```

**Resource-Constrained Nodes**

```
// Optimize for lower memory usage
LegacyPoolConfig: &legacypool.Config{
    AccountSlots: 8,     // Fewer transactions per account
    GlobalSlots:  2048,  // Lower global capacity
    AccountQueue: 32,    // Smaller queues
    GlobalQueue:  512,
    Lifetime:     1*time.Hour, // Shorter retention
}
```

**DeFi-Optimized Configuration**

```
// Balance throughput with MEV protection
LegacyPoolConfig: &legacypool.Config{
    AccountSlots: 32,
    GlobalSlots:  8192,
    PriceBump:    25,    // Higher replacement threshold
    Lifetime:     30*time.Minute, // Faster eviction for MEV
}
```

### State Management <a href="#state-management" id="state-management"></a>

The mempool maintains transaction state through the unified `ExperimentalEVMMempool` structure, which manages separate pools for EVM and Ontomir transactions while providing a single interface. This experimental implementation handles fee-based prioritization, nonce sequencing, and transaction verification through an integrated ante handler.

### Testing <a href="#testing" id="testing"></a>

The mempool behavior can be verified using the test scripts provided in the [Ontomir/evm](https://github.com/Ontomir/evm) repository. The [`tests/systemtests/Counter/script/SimpleSends.s.sol`](https://github.com/Ontomir/evm/blob/v0.4.1/tests/systemtests/Counter/script/SimpleSends.s.sol) script demonstrates typical Ethereum tooling behavior - it sends 10 sequential transactions in a batch, which naturally arrive out of order and create nonce gaps.

### Integration <a href="#integration-1" id="integration-1"></a>

For step-by-step instructions on integrating the EVM mempool into your chain, see the EVM Mempool Integration guide.
