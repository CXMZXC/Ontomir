# EVM Mempool Integration

> Integrate the EVM mempool with your Ontomir EVM chain

### Overview <a href="#overview" id="overview"></a>

This guide provides step-by-step instructions for integrating the EVM mempool into your Ontomir EVM chain. For conceptual information about mempool design and architecture, see the Mempool Concepts page.

#### Step 1: Add EVM Mempool to App Struct <a href="#step-1-add-evm-mempool-to-app-struct" id="step-1-add-evm-mempool-to-app-struct"></a>

Update your `app/app.go` to include the EVM mempool:

```
type App struct {
    *baseapp.BaseApp
    // ... other keepers

    // Ontomir EVM keepers
    FeeMarketKeeper   feemarketkeeper.Keeper
    EVMKeeper         *evmkeeper.Keeper
    EVMMempool        *evmmempool.ExperimentalEVMMempool
}
```

#### Step 2: Configure Mempool in NewApp Constructor <a href="#step-2-configure-mempool-in-newapp-constructor" id="step-2-configure-mempool-in-newapp-constructor"></a>

The mempool must be initialized \*\*after\*\* the antehandler has been set in the app.

Add the following configuration in your `NewApp` constructor:

\`\`\`go // Set the EVM priority nonce mempool if evmtypes.GetChainConfig() != nil { mempoolConfig := \&evmmempool.EVMMempoolConfig{ AnteHandler: app.GetAnteHandler(), BlockGasLimit: 100\_000\_000, }

```
  evmMempool := evmmempool.NewExperimentalEVMMempool(
      app.CreateQueryContext,
      logger,
      app.EVMKeeper,
      app.FeeMarketKeeper,
      app.txConfig,
      app.clientCtx,
      mempoolConfig,
  )
  app.EVMMempool = evmMempool

  // Replace BaseApp mempool
  app.SetMempool(evmMempool)

  // Set custom CheckTx handler for nonce gap support
  checkTxHandler := evmmempool.NewCheckTxHandler(evmMempool)
  app.SetCheckTxHandler(checkTxHandler)

  // Set custom PrepareProposal handler
  abciProposalHandler := baseapp.NewDefaultProposalHandler(evmMempool, app)
  abciProposalHandler.SetSignerExtractionAdapter(
      evmmempool.NewEthSignerExtractionAdapter(
          sdkmempool.NewDefaultSignerExtractionAdapter(),
      ),
  )
  app.SetPrepareProposal(abciProposalHandler.PrepareProposalHandler())
```

}

````
</Expandable>

<Warning>
**Breaking Change from v0.4.x:** The global mempool registry (`SetGlobalEVMMempool`) has been removed. Mempool is now passed directly to the JSON-RPC server during initialization.
</Warning>

## Configuration Options

The `EVMMempoolConfig` struct provides several configuration options for customizing the mempool behavior:

### Minimal Configuration

```go
mempoolConfig := &evmmempool.EVMMempoolConfig{
  AnteHandler:   app.GetAnteHandler(),
  BlockGasLimit: 100_000_000, // 100M gas limit
}
````

#### Full Configuration Options <a href="#full-configuration-options" id="full-configuration-options"></a>

```
type EVMMempoolConfig struct {
    // Required: AnteHandler for transaction validation
    AnteHandler   sdk.AnteHandler

    // Required: Block gas limit for transaction selection
    BlockGasLimit uint64

    // Optional: Custom legacy pool configuration (replaces TxPool)
    LegacyPoolConfig *legacypool.Config

    // Optional: Custom Ontomir pool configuration (replaces OntomirPool)
    OntomirPoolConfig *sdkmempool.PriorityNonceMempoolConfig[math.Int]

    // Optional: Custom broadcast function for promoted transactions
    BroadcastTxFn func(txs []*ethtypes.Transaction) error

    // Optional: Minimum tip required for EVM transactions
    MinTip *uint256.Int
}
```

**Defaults and Fallbacks**

* If `BlockGasLimit` is `0`, the mempool uses a fallback of `100_000_000` gas.
* If `OntomirPoolConfig` is not provided, a default `PriorityNonceMempool` is created with:
  * Priority = `(fee_amount / gas_limit)` in the chain bond denom
  * Comparator = big-int comparison (higher is selected first)
  * `MinValue = 0`
* If `BroadcastTxFn` is not provided, a default is created that uses the app `clientCtx`/`txConfig` to broadcast EVM transactions when they are promoted from queued → pending.
* `MinTip` is optional. If unset, selection uses the effective tip from each tx (`min(gas_tip_cap, gas_fee_cap - base_fee)`).

#### v0.4.x to v0.5.0 Migration <a href="#v04x-to-v050-migration" id="v04x-to-v050-migration"></a>

**Breaking Change:** Pre-built pools replaced with configuration objects

**Before (v0.4.x):**

```
mempoolConfig := &evmmempool.EVMMempoolConfig{
    TxPool:     customTxPool,      // ← REMOVED
    OntomirPool: customOntomirPool,  // ← REMOVED
    AnteHandler: app.GetAnteHandler(),
    BlockGasLimit: 100_000_000,
}
```

**After (v0.5.0):**

```
mempoolConfig := &evmmempool.EVMMempoolConfig{
    LegacyPoolConfig: &legacypool.Config{     // ← NEW
        AccountSlots: 16,
        GlobalSlots:  5120,
        PriceLimit:   1,
        // ... other config options
    },
    OntomirPoolConfig: &sdkmempool.PriorityNonceMempoolConfig[math.Int]{ // ← NEW
        TxPriority: customPriorityConfig,
    },
    AnteHandler:   app.GetAnteHandler(),
    BlockGasLimit: 100_000_000,
}
```

#### Custom Ontomir Mempool Configuration <a href="#custom-ontomir-mempool-configuration" id="custom-ontomir-mempool-configuration"></a>

The mempool uses a `PriorityNonceMempool` for Ontomir transactions by default. You can customize the priority calculation:

```
// Define custom priority calculation for Ontomir transactions
priorityConfig := sdkmempool.PriorityNonceMempoolConfig[math.Int]{
    TxPriority: sdkmempool.TxPriority[math.Int]{
        GetTxPriority: func(goCtx context.Context, tx sdk.Tx) math.Int {
            feeTx, ok := tx.(sdk.FeeTx)
            if !ok {
                return math.ZeroInt()
            }

            // Get fee in bond denomination
            bondDenom := "test" // or your chain's bond denom
            fee := feeTx.GetFee()
            found, coin := fee.Find(bondDenom)
            if !found {
                return math.ZeroInt()
            }

            // Calculate gas price: fee_amount / gas_limit
            gasPrice := coin.Amount.Quo(math.NewIntFromUint64(feeTx.GetGas()))
            return gasPrice
        },
        Compare: func(a, b math.Int) int {
            return a.BigInt().Cmp(b.BigInt()) // Higher values have priority
        },
        MinValue: math.ZeroInt(),
    },
}

mempoolConfig := &evmmempool.EVMMempoolConfig{
    AnteHandler:      app.GetAnteHandler(),
    BlockGasLimit:    100_000_000,
    OntomirPoolConfig: &priorityConfig, // Pass config instead of pre-built pool
}
```

#### Custom Block Gas Limit <a href="#custom-block-gas-limit" id="custom-block-gas-limit"></a>

Different chains may require different gas limits based on their capacity:

```
// Example: 50M gas limit for lower capacity chains
mempoolConfig := &evmmempool.EVMMempoolConfig{
    AnteHandler:   app.GetAnteHandler(),
    BlockGasLimit: 50_000_000,
}
```

#### Event Bus Integration <a href="#event-bus-integration" id="event-bus-integration"></a>

For best results, connect the mempool to CometBFT's EventBus so it can react to finalized blocks:

```
// After starting the CometBFT node
if m, ok := app.GetMempool().(*evmmempool.ExperimentalEVMMempool); ok {
    m.SetEventBus(bftNode.EventBus())
}
```

This enables chain-head notifications so the mempool can promptly promote/evict transactions when blocks are committed.

### Architecture Components <a href="#architecture-components" id="architecture-components"></a>

The EVM mempool consists of several key components:

#### ExperimentalEVMMempool <a href="#experimentalevmmempool" id="experimentalevmmempool"></a>

The main coordinator implementing Ontomir SDK's `ExtMempool` interface (`mempool/mempool.go`).

**Key Methods**:

* `Insert(ctx, tx)`: Routes transactions to appropriate pools
* `Select(ctx, filter)`: Returns unified iterator over all transactions
* `Remove(tx)`: Handles transaction removal with EVM-specific logic
* `InsertInvalidNonce(txBytes)`: Queues nonce-gapped EVM transactions locally

#### CheckTx Handler <a href="#checktx-handler" id="checktx-handler"></a>

Custom transaction validation that handles nonce gaps specially (`mempool/check_tx.go`).

**Special Handling**: On `ErrNonceGap` for EVM transactions:

```
if errors.Is(err, ErrNonceGap) {
    // Route to local queue instead of rejecting
    err := mempool.InsertInvalidNonce(request.Tx)
    // Must intercept error and return success to EVM client
    return interceptedSuccess
}
```

#### TxPool <a href="#txpool" id="txpool"></a>

Direct port of Ethereum's transaction pool managing both pending and queued transactions (`mempool/txpool/`).

**Key Features**:

* Uses `vm.StateDB` interface for Ontomir state compatibility
* Implements `BroadcastTxFn` callback for transaction promotion
* Ontomir-specific reset logic for instant finality

#### PriorityNonceMempool <a href="#prioritynoncemempool" id="prioritynoncemempool"></a>

Standard Ontomir SDK mempool for non-EVM transactions with fee-based prioritization.

**Default Priority Calculation**:

```
// Calculate effective gas price
priority = (fee_amount / gas_limit) - base_fee
```

### Transaction Type Routing <a href="#transaction-type-routing" id="transaction-type-routing"></a>

The mempool handles different transaction types appropriately:

#### Ethereum Transactions (MsgEthereumTx) <a href="#ethereum-transactions-msgethereumtx" id="ethereum-transactions-msgethereumtx"></a>

* **Tier 1 (Local)**: EVM TxPool handles nonce gaps and promotion
* **Tier 2 (Network)**: CometBFT broadcasts executable transactions

#### Ontomir Transactions (Bank, Staking, Gov, etc.) <a href="#ontomir-transactions-bank-staking-gov-etc" id="ontomir-transactions-bank-staking-gov-etc"></a>

* **Direct to Tier 2**: Always go directly to CometBFT mempool
* **Standard Flow**: Follow normal Ontomir SDK validation and broadcasting
* **Priority-Based**: Use PriorityNonceMempool for fee-based ordering

### Unified Transaction Selection <a href="#unified-transaction-selection" id="unified-transaction-selection"></a>

During block building, both transaction types compete fairly based on their effective tips:

```
// Simplified selection logic
func SelectTransactions() Iterator {
    evmTxs := GetPendingEVMTransactions()      // From local TxPool
    OntomirTxs := GetPendingOntomirTransactions() // From Ontomir mempool

    return NewUnifiedIterator(evmTxs, OntomirTxs) // Fee-based priority
}
```

**Fee Comparison**:

* **EVM**: `gas_tip_cap` or `min(gas_tip_cap, gas_fee_cap - base_fee)`
* **Ontomir**: `(fee_amount / gas_limit) - base_fee`
* **Selection**: Higher effective tip gets selected first

### Testing Your Integration <a href="#testing-your-integration" id="testing-your-integration"></a>

#### Verify Nonce Gap Handling <a href="#verify-nonce-gap-handling" id="verify-nonce-gap-handling"></a>

Test that transactions with nonce gaps are properly queued:

```
// Send transactions out of order
await wallet.sendTransaction({nonce: 100, ...}); // OK: Immediate execution
await wallet.sendTransaction({nonce: 102, ...}); // OK: Queued locally (gap)
await wallet.sendTransaction({nonce: 101, ...}); // OK: Fills gap, both execute
```

#### Test Transaction Replacement <a href="#test-transaction-replacement" id="test-transaction-replacement"></a>

Verify that higher-fee transactions replace lower-fee ones:

```
// Send initial transaction
const tx1 = await wallet.sendTransaction({
  nonce: 100,
  gasPrice: parseUnits("20", "gwei")
});

// Replace with higher fee
const tx2 = await wallet.sendTransaction({
  nonce: 100, // Same nonce
  gasPrice: parseUnits("30", "gwei") // Higher fee
});
// tx1 is replaced by tx2
```

#### Verify Batch Deployments <a href="#verify-batch-deployments" id="verify-batch-deployments"></a>

Test typical deployment scripts (like Uniswap) that send many transactions at once:

```
// Deploy multiple contracts in quick succession
const factory = await Factory.deploy();
const router = await Router.deploy(factory.address);
const multicall = await Multicall.deploy();
// All transactions should queue and execute properly
```

### Monitoring and Debugging <a href="#monitoring-and-debugging" id="monitoring-and-debugging"></a>

Use the [txpool RPC methods](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool-methods) to monitor mempool state:

* `txpool_status`: Get pending and queued transaction counts
* `txpool_content`: View all transactions in the pool
* `txpool_inspect`: Get human-readable transaction summaries
* `txpool_contentFrom`: View transactions from specific addresses
* Mempool Concepts - Understanding mempool behavior and design
* EVM Module Integration - Prerequisites for mempool integration
* [JSON-RPC Methods](https://about/docs/evm/next/api-reference/ethereum-json-rpc/methods#txpool-methods) - Mempool query methods
