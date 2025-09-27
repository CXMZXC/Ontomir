# Feemarket

> EIP-1559 modualarized

The feemarket module from [Ontomir/evm](https://github.com/Ontomir/evm) implements [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) dynamic fee pricing for Ontomir SDK chains with EVM compatibility.

This is part of Ontomir/evm, not standard Ontomir SDK. For conceptual understanding of EIP-1559, see \[EIP-1559 Fee Market]\(/docs/evm/next/documentation/concepts/eip-1559-feemarket).

### Parameters <a href="#parameters" id="parameters"></a>

The module contains the following parameters ([types](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/types/params.go), [proto](https://github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/feemarket.proto)):

| Parameter                  | Type    | Default      | Description                                                   |
| -------------------------- | ------- | ------------ | ------------------------------------------------------------- |
| `NoBaseFee`                | bool    | `false`      | Disables EIP-1559 base fee (enables legacy pricing)           |
| `BaseFeeChangeDenominator` | uint32  | `8`          | Controls max base fee change per block (higher = more stable) |
| `ElasticityMultiplier`     | uint32  | `2`          | Determines target block utilization (2 = 50% target)          |
| `EnableHeight`             | int64   | `0`          | Block height to activate fee market (0 = genesis)             |
| `BaseFee`                  | sdk.Dec | `1000000000` | Current base fee (auto-adjusts each block)                    |
| `MinGasPrice`              | sdk.Dec | `0`          | Minimum gas price floor (cannot go below)                     |
| `MinGasMultiplier`         | sdk.Dec | `0.5`        | Minimum gas used multiplier for gas wanted                    |

#### Parameter Details <a href="#parameter-details" id="parameter-details"></a>

Controls fee volatility by limiting the maximum change per block:

| Value | Max Change/Block | Blocks to 10x | Use Case                        |
| ----- | ---------------- | ------------- | ------------------------------- |
| 4     | ±25%             | \~10 blocks   | Testnets, rapid price discovery |
| **8** | ±12.5%           | \~20 blocks   | Ethereum standard, balanced     |
| 16    | ±6.25%           | \~40 blocks   | Stable fees, production chains  |
| 32    | ±3.125%          | \~80 blocks   | Very stable, enterprise chains  |
|       |                  |               |                                 |

Sets the target block utilization:

| Value | Target % | Behavior                      | Recommended For               |
| ----- | -------- | ----------------------------- | ----------------------------- |
| 1     | 100%     | Only increases at full blocks | Maximum throughput chains     |
| **2** | 50%      | Balanced (Ethereum standard)  | General purpose chains        |
| 4     | 25%      | Aggressive fee increases      | Anti-spam focused chains      |
| 10    | 10%      | Very aggressive pricing       | High-value transaction chains |
|       |          |                               |                               |

Sets the absolute price floor:

```
BaseFee = max(CalculatedBaseFee, MinGasPrice)
```

This prevents fees from dropping too low during periods of low activity, maintaining spam resistance.

#### Parameter Storage Format <a href="#parameter-storage-format" id="parameter-storage-format"></a>

`sdk.Dec` (LegacyDec) values are stored as strings with 18 decimal precision ([Ontomir-sdk](https://github.com/Ontomir/Ontomir-sdk/blob/main/math/dec.go)):

| Human Value | Stored String                  | Formula      |
| ----------- | ------------------------------ | ------------ |
| 1 unit      | "1000000000000000000"          | 1 × 10^18    |
| 0.5 (50%)   | "500000000000000000"           | 0.5 × 10^18  |
| 1 billion   | "1000000000000000000000000000" | 10^9 × 10^18 |

### Configuration <a href="#configuration" id="configuration"></a>

Adjust parameters to achieve specific network behaviors ([params validation](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/types/params.go#L43-L142)):

| **Goal**                | **Parameter**                 | **Adjustment** | **Effect**                               |
| ----------------------- | ----------------------------- | -------------- | ---------------------------------------- |
| Faster fee response     | `base_fee_change_denominator` | 8 → 4          | 2x faster adjustments (±25% vs ±12.5%)   |
| More stable fees        | `base_fee_change_denominator` | 8 → 16         | 2x slower adjustments (±6.25% vs ±12.5%) |
| Earlier fee increases   | `elasticity_multiplier`       | 2 → 4          | Fees rise at 25% vs 50% utilization      |
| Stronger spam deterrent | `min_gas_price`               | Low → High     | Higher transaction cost floor            |

#### Configuration Profiles <a href="#configuration-profiles" id="configuration-profiles"></a>

\`\`\`json Ethereum-Compatible expandable { "no\_base\_fee": false, "base\_fee\_change\_denominator": 8, // ±12.5% max per block "elasticity\_multiplier": 2, // 50% target utilization "enable\_height": 0, // Active from genesis "base\_fee": "1000000000000000000000", // Starting base fee "min\_gas\_price": "1000000000000000000", // Price floor "min\_gas\_multiplier": "500000000000000000" // 0.5 (anti-manipulation) } // Behavior: Standard Ethereum, balanced for general use // 100k gas tx: \~0.1 tokens at base, up to 10x during congestion \`\`\`

```
{
  "no_base_fee": false,
  "base_fee_change_denominator": 16,     // ±6.25% max per block
  "elasticity_multiplier": 2,            // 50% target utilization
  "enable_height": 0,
  "base_fee": "100000000000000000000",   // Lower starting fee
  "min_gas_price": "10000000000000000",  // Lower floor
  "min_gas_multiplier": "500000000000000000"
}
// Behavior: Slow, predictable changes for DeFi/DEX chains
// 100k gas tx: ~0.01 tokens, gradual increases
```

```
{
  "no_base_fee": false,
  "base_fee_change_denominator": 4,      // ±25% max per block
  "elasticity_multiplier": 4,            // 25% target utilization
  "enable_height": 0,
  "base_fee": "10000000000000000000000",  // High starting fee
  "min_gas_price": "10000000000000000000000", // High floor
  "min_gas_multiplier": "1000000000000000000" // 1.0 (strict)
}
// Behavior: Rapid adjustments, high minimum for premium chains
// 100k gas tx: ~1 token minimum, aggressive scaling
```

#### Scenario Comparisons <a href="#scenario-comparisons" id="scenario-comparisons"></a>

\| Config | 5 Blocks | 10 Blocks | 10x Time | | ------------------------- | -------- | --------- | ----------- | | \*\*Ethereum\*\* (d=8, e=2) | +34% | +79% | \\\~20 blocks | | \*\*Stable\*\* (d=16, e=2) | +16% | +34% | \\\~40 blocks | | \*\*Aggressive\*\* (d=4, e=4) | +101% | +304% | \\\~8 blocks | | Config | 5 Blocks | 10 Blocks | Floor | | -------------- | -------- | --------- | ----------- | | \*\*Ethereum\*\* | -26% | -45% | \\\~15 blocks | | \*\*Stable\*\* | -14% | -26% | \\\~30 blocks | | \*\*Aggressive\*\* | -61% | -85% | \\\~6 blocks |

#### Decimal Precision Support <a href="#decimal-precision-support" id="decimal-precision-support"></a>

The module supports different token decimal configurations:

| Decimals | Conversion Factor | Min Unit Gas | Example       |
| -------- | ----------------- | ------------ | ------------- |
| 6        | 10^12             | 10^-12       | USDC          |
| 8        | 10^10             | 10^-10       | BTC-wrapped   |
| 18       | 1                 | 1            | ETH, most EVM |

### Configuration In Production <a href="#configuration-in-production" id="configuration-in-production"></a>

Parameters can be updated through governance proposals ([msg server](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/msg_server.go)):

```
// MsgUpdateParams structure
// Source: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/tx.proto
{
  "@type": "/Ontomir.evm.feemarket.v1.MsgUpdateParams",
  "authority": "Ontomir10d07y265gmmuvt4z0w9aw880jnsr700j6zn9kn",
  "params": {
    "no_base_fee": false,
    "base_fee_change_denominator": 16,
    "elasticity_multiplier": 2,
    "enable_height": 100000,
    "base_fee": "1000000000000000000000",
    "min_gas_price": "1000000000000000000",
    "min_gas_multiplier": "500000000000000000"
  }
}
```

Submit with:

```
# Submit governance proposal
# Source: github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/msg_server.go#L17-L39
evmd tx gov submit-proposal update-feemarket-params.json \
  --from <key> \
  --deposit 1000000000test
```

### Best Practices <a href="#best-practices" id="best-practices"></a>

#### Parameter Selection by Use Case <a href="#parameter-selection-by-use-case" id="parameter-selection-by-use-case"></a>

| **Chain Type**         | **Denominator** | **Elasticity** | **MinGasPrice** | **Why**                                |
| ---------------------- | --------------- | -------------- | --------------- | -------------------------------------- |
| Testing/Development    | 8               | 2              | Zero/Low        | Quick iteration                        |
| Production General     | 8-16            | 2              | Medium          | Balance stability and responsiveness   |
| High-Frequency Trading | 16-32           | 1-2            | Low             | Predictable fees, high throughput      |
| Enterprise/Private     | 32              | 2-3            | High            | Maximum stability, cost predictability |
| Gaming/Micro-tx        | 4-8             | 1              | Very Low        | Fast adjustments, minimal fees         |

#### Block Time Correlation <a href="#block-time-correlation" id="block-time-correlation"></a>

Faster blocks require higher denominators to maintain stability:

```
Recommended Denominator = 8 × (6s / YourBlockTime)
```

Examples:

* 1s blocks → denominator ≥16 (prevents wild swings)
* 6s blocks → denominator = 8 (Ethereum standard)
* 12s blocks → denominator = 4 (can be more responsive)

#### Monitoring Key Metrics <a href="#monitoring-key-metrics" id="monitoring-key-metrics"></a>

| **Metric**       | **Formula**                  | **Healthy** | **Action if Unhealthy** |
| ---------------- | ---------------------------- | ----------- | ----------------------- |
| Fee Volatility   | `σ(fees)/μ(fees)`            | <20% daily  | Increase denominator    |
| Utilization Rate | `gasUsed/maxGas`             | 40-60% avg  | Adjust elasticity       |
| Floor Hit Rate   | `count(fee==min)\/blocks`    | <10%        | Lower min\_gas\_price   |
| Recovery Speed   | Blocks to return to baseline | 20-40       | Tune denominator        |

#### Integration Examples <a href="#integration-examples" id="integration-examples"></a>

\`\`\`javascript Wallet-Integration expandable // Get current base fee from latest block const block = await provider.getBlock('latest'); const baseFee = block.baseFeePerGas;

// Add buffer for inclusion certainty (20% buffer) const maxFeePerGas = baseFee \* 120n / 100n; const maxPriorityFeePerGas = ethers.parseUnits("2", "gwei");

// Send EIP-1559 transaction const tx = await wallet.sendTransaction({ type: 2, maxFeePerGas, maxPriorityFeePerGas, to: "0x...", value: ethers.parseEther("1.0"), gasLimit: 21000 });

````

```javascript Fee-Monitoring expandable
// Get current gas price (includes base fee)
const gasPrice = await provider.send('eth_gasPrice', []);
console.log('Current gas price:', ethers.formatUnits(gasPrice, 'gwei'), 'gwei');

// Get fee history
const feeHistory = await provider.send('eth_feeHistory', [
  '0x10', // last 16 blocks
  'latest',
  [25, 50, 75]
]);

// Process base fees from history
const baseFees = feeHistory.baseFeePerGas.map(fee =>
  ethers.formatUnits(fee, 'gwei')
);
console.log('Base fee history:', baseFees);

// Check if fees are elevated
const currentBaseFee = BigInt(feeHistory.baseFeePerGas[feeHistory.baseFeePerGas.length - 1]);
const threshold = ethers.parseUnits('100', 'gwei');
const isHighFee = currentBaseFee > threshold;
````

### Interfaces <a href="#interfaces" id="interfaces"></a>

#### EVM JSON-RPC <a href="#evm-json-rpc" id="evm-json-rpc"></a>

The fee market integrates with standard Ethereum JSON-RPC methods ([RPC backend](https://github.com/Ontomir/evm/blob/v0.4.1/rpc/backend/chain_info.go)):

Returns the current base fee as hex string. In EIP-1559 mode, returns the base fee; in legacy mode, returns the median gas price.

```
// Request
{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":1}

// Response
{"jsonrpc":"2.0","id":1,"result":"0x3b9aca00"} // 1 gwei in hex
```

Returns suggested priority fee (tip) per gas. In Ontomir/evm with the experimental mempool, transactions are prioritized by effective gas tip (fees), with ties broken by arrival time.

```
// Request
{"jsonrpc":"2.0","method":"eth_maxPriorityFeePerGas","params":[],"id":1}

// Response
{"jsonrpc":"2.0","id":1,"result":"0x77359400"} // Suggested tip in wei
```

Returns base fee and gas utilization history for a range of blocks. Number of blocks to return (hex or decimal)

```
<ParamField body="newestBlock" type="string" required>
  Latest block to include ("latest", "pending", or block number)
</ParamField>

<ParamField body="rewardPercentiles" type="array">
  Array of percentiles for priority fee calculations
</ParamField>
```

```
// Request
{
  "jsonrpc":"2.0",
  "method":"eth_feeHistory",
  "params":["0x5", "latest", [25, 50, 75]],
  "id":1
}

// Response
{
  "jsonrpc":"2.0",
  "id":1,
  "result":{
    "baseFeePerGas": [
      "0x3b9aca00",
      "0x3c9aca00",
      "0x3d9aca00",
      "0x3e9aca00",
      "0x3f9aca00"
    ],
    "gasUsedRatio": [0.5, 0.55, 0.6, 0.48, 0.52],
    "oldestBlock": "0x1234",
    "reward": [
      ["0x0", "0x0", "0x0"],
      ["0x0", "0x0", "0x0"],
      ["0x0", "0x0", "0x0"],
      ["0x0", "0x0", "0x0"],
      ["0x0", "0x0", "0x0"]
    ]
  }
}
```

Block responses include \`baseFeePerGas\` field when EIP-1559 is enabled.

```
{
  "jsonrpc":"2.0",
  "id":1,
  "result":{
    "number": "0x1234",
    "hash": "0x...",
    "baseFeePerGas": "0x3b9aca00", // Base fee for this block
    // ... other block fields
  }
}
```

**Transaction Types**

The fee market supports both legacy and EIP-1559 transaction types ([types](https://github.com/Ontomir/evm/blob/v0.4.1/x/vm/types/tx.go)):

\`\`\`json Legacy-Transaction { "type": "0x0", "gasPrice": "0x3b9aca00", // Must be >= current base fee "gas": "0x5208", "to": "0x...", "value": "0x...", "data": "0x..." } \`\`\`

```
{
  "type": "0x2",
  "maxFeePerGas": "0x3b9aca00",      // Maximum total fee willing to pay
  "maxPriorityFeePerGas": "0x77359400", // Tip that affects transaction priority in mempool
  "gas": "0x5208",
  "to": "0x...",
  "value": "0x...",
  "data": "0x..."
}
```

#### CLI <a href="#cli" id="cli"></a>

\`\`\`bash Query-Parameters # Query module parameters # Source: github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/client/cli/query.go evmd query feemarket params \`\`\`

```
# Query current base fee
# Source: github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/client/cli/query.go
evmd query feemarket base-fee
```

```
# Query block gas wanted
# Source: github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/client/cli/query.go
evmd query feemarket block-gas
```

#### gRPC <a href="#grpc" id="grpc"></a>

\`\`\`proto Query-Parameters expandable // Request all module parameters // Proto: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/query.proto grpcurl -plaintext localhost:9090 feemarket.Query/Params

// Response { "params": { "no\_base\_fee": false, "base\_fee\_change\_denominator": 8, "elasticity\_multiplier": 2, "enable\_height": 0, "base\_fee": "1000000000000000000000", "min\_gas\_price": "1000000000000000000", "min\_gas\_multiplier": "500000000000000000" } }

````

```proto Query-Base-Fee
// Request current base fee
// Proto: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/query.proto
grpcurl -plaintext localhost:9090 feemarket.Query/BaseFee

// Response
{
  "base_fee": "1000000000000000000000"
}
````

```
// Request block gas wanted
// Proto: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/query.proto
grpcurl -plaintext localhost:9090 feemarket.Query/BlockGas

// Response
{
  "gas": "5000000"
}
```

#### REST <a href="#rest" id="rest"></a>

\`\`\`http Query-Parameters expandable # REST endpoint for parameters # Proto: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/query.proto GET /Ontomir/feemarket/v1/params

// Response { "params": { "no\_base\_fee": false, "base\_fee\_change\_denominator": 8, "elasticity\_multiplier": 2, "enable\_height": 0, "base\_fee": "1000000000000000000000", "min\_gas\_price": "1000000000000000000", "min\_gas\_multiplier": "500000000000000000" } }

````

```http Query-Base-Fee
# REST endpoint for base fee
# Proto: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/query.proto
GET /Ontomir/feemarket/v1/base_fee

// Response
{
  "base_fee": "1000000000000000000000"
}
````

```
# REST endpoint for block gas
# Proto: github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/query.proto
GET /Ontomir/feemarket/v1/block_gas

// Response
{
  "gas": "5000000"
}
```

### Technical Implementation <a href="#technical-implementation" id="technical-implementation"></a>

#### State <a href="#state" id="state"></a>

The module maintains the following state ([genesis proto](https://github.com/Ontomir/evm/blob/v0.4.1/proto/Ontomir/evm/feemarket/v1/genesis.proto)):

**State Objects**

| State Object   | Description                     | Key              | Value            | Store     |
| -------------- | ------------------------------- | ---------------- | ---------------- | --------- |
| BlockGasWanted | Gas wanted in the current block | `[]byte{1}`      | `[]byte{uint64}` | Transient |
| BaseFee        | Current base fee                | `Params.BaseFee` | `sdk.Dec`        | KV        |
| Params         | Module parameters               | `[]byte{0x00}`   | `Params`         | KV        |

**Genesis State**

```
type GenesisState struct {
    Params Params
    BaseFee sdk.Dec
}
```

#### Begin Block <a href="#begin-block" id="begin-block"></a>

At the beginning of each block, the module calculates and sets the new base fee ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/abci.go#L16-L45)):

```
func (k Keeper) BeginBlock(ctx sdk.Context) error {
    baseFee := k.CalculateBaseFee(ctx)

    // Skip if base fee is nil (not enabled)
    if baseFee.IsNil() {
        return nil
    }

    // Update the base fee in state
    k.SetBaseFee(ctx, baseFee)

    // Emit event with new base fee
    ctx.EventManager().EmitEvent(
        sdk.NewEvent(
            "fee_market",
            sdk.NewAttribute("base_fee", baseFee.String()),
        ),
    )
    return nil
}
```

#### End Block <a href="#end-block" id="end-block"></a>

At the end of each block, the module updates the gas wanted for base fee calculation ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/abci.go#L47-L92)):

```
func (k Keeper) EndBlock(ctx sdk.Context) error {
    gasWanted := k.GetTransientGasWanted(ctx)
    gasUsed := ctx.BlockGasMeter().GasConsumedToLimit()

    // Apply MinGasMultiplier to prevent manipulation
    // gasWanted = max(gasWanted * MinGasMultiplier, gasUsed)
    minGasMultiplier := k.GetParams(ctx).MinGasMultiplier
    limitedGasWanted := NewDec(gasWanted).Mul(minGasMultiplier)
    updatedGasWanted := MaxDec(limitedGasWanted, NewDec(gasUsed)).TruncateInt()

    // Store for next block's base fee calculation
    k.SetBlockGasWanted(ctx, updatedGasWanted)

    ctx.EventManager().EmitEvent(
        sdk.NewEvent(
            "block_gas",
            sdk.NewAttribute("height", fmt.Sprintf("%d", ctx.BlockHeight())),
            sdk.NewAttribute("amount", fmt.Sprintf("%d", updatedGasWanted)),
        ),
    )
    return nil
}
```

#### Base Fee Calculation Algorithm <a href="#base-fee-calculation-algorithm" id="base-fee-calculation-algorithm"></a>

The calculation follows the EIP-1559 specification with Ontomir SDK adaptations ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/eip1559.go#L18-L69), [utils](https://github.com/Ontomir/evm/blob/v0.4.1/utils/utils.go#L229-L254)):

\`\`\`go func CalcGasBaseFee(gasUsed, gasTarget, baseFeeChangeDenom uint64, baseFee, minUnitGas, minGasPrice LegacyDec) LegacyDec { // No change if at target if gasUsed == gasTarget { return baseFee }

```
  // Calculate adjustment magnitude
  delta := abs(gasUsed - gasTarget)
  adjustment := baseFee.Mul(delta).Quo(gasTarget).Quo(baseFeeChangeDenom)

  if gasUsed > gasTarget {
      // Increase base fee (minimum 1 unit)
      baseFeeDelta := MaxDec(adjustment, minUnitGas)
      return baseFee.Add(baseFeeDelta)
  } else {
      // Decrease base fee (not below minGasPrice)
      return MaxDec(baseFee.Sub(adjustment), minGasPrice)
  }
```

}

````

The `minUnitGas` is calculated as `1 / 10^decimals` to ensure minimum increments based on token precision ([source](https://github.com/Ontomir/evm/blob/v0.4.1/utils/utils.go#L229-L254)).
</Expandable>

### Keeper

The keeper manages fee market state and calculations ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/keeper.go)):

| Method                                        | Description           | Usage                               |
| --------------------------------------------- | --------------------- | ----------------------------------- |
| `GetBaseFee()` / `SetBaseFee()`               | Current base fee      | Read/update during block processing |
| `GetParams()` / `SetParams()`                 | Module parameters     | Governance updates                  |
| `GetBlockGasWanted()` / `SetBlockGasWanted()` | Gas tracking          | Base fee calculation                |
| `CalculateBaseFee()`                          | Compute next base fee | Called in BeginBlock                |

The module also implements `PostTxProcessing` hooks for EVM transaction processing ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/hooks.go)).

### Events

The module emits the following events ([types](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/types/events.go)):

#### Fee Market Block Fees

| Attribute Key | Attribute Value           |
| ------------- | ------------------------- |
| height        | Current block height      |
| base\_fee     | New base fee              |
| gas\_wanted   | Total gas wanted in block |
| gas\_used     | Total gas used in block   |

Example:

```json
{
"type": "fee_market_block_fees",
"attributes": [
  {"key": "height", "value": "12345"},
  {"key": "base_fee", "value": "1000000000"},
  {"key": "gas_wanted", "value": "5000000"},
  {"key": "gas_used", "value": "4800000"}
]
}
````

#### AnteHandlers <a href="#antehandlers" id="antehandlers"></a>

The module integrates with EVM transactions through the `EVMMonoDecorator`, which performs all fee-related validations in a single pass ([source](https://github.com/Ontomir/evm/blob/v0.4.1/ante/evm/mono_decorator.go#L50-L252)):

**EVMMonoDecorator**

The primary ante handler that validates and processes EVM transaction fees ([mono\_decorator.go](https://github.com/Ontomir/evm/blob/v0.4.1/ante/evm/mono_decorator.go)):

\`\`\`go // 1. Check mempool minimum fee (for pre-London transactions) if ctx.IsCheckTx() && !simulate && !isLondon { requiredFee := mempoolMinGasPrice.Mul(gasLimit) if fee.LT(requiredFee) { return ErrInsufficientFee } }

// 2. Calculate effective fee for EIP-1559 transactions if ethTx.Type() >= DynamicFeeTxType && baseFee != nil { feeAmt = ethMsg.GetEffectiveFee(baseFee) fee = NewDecFromBigInt(feeAmt) }

// 3. Check global minimum gas price if globalMinGasPrice.IsPositive() { requiredFee := globalMinGasPrice.Mul(gasLimit) if fee.LT(requiredFee) { return ErrInsufficientFee } }

// 4. Track gas wanted for base fee calculation if isLondon && baseFeeEnabled { feeMarketKeeper.AddTransientGasWanted(ctx, gasWanted) }

````
</Expandable>

#### Fee Validation Functions

##### CheckMempoolFee

Validates transaction fees against the local mempool minimum ([source](https://github.com/Ontomir/evm/blob/v0.4.1/ante/evm/02_mempool_fee.go#L13-L29)):

<Expandable title="Implementation details">
```go
func CheckMempoolFee(fee, mempoolMinGasPrice, gasLimit LegacyDec, isLondon bool) error {
    // Skip for London-enabled chains (EIP-1559 handles this)
    if isLondon {
        return nil
    }

    requiredFee := mempoolMinGasPrice.Mul(gasLimit)
    if fee.LT(requiredFee) {
        return ErrInsufficientFee
    }
    return nil
}
````

**Note**: This check is bypassed for London-enabled chains since EIP-1559 base fee mechanism handles minimum pricing.

**CheckGlobalFee**

Enforces the chain-wide minimum gas price set via governance ([source](https://github.com/Ontomir/evm/blob/v0.4.1/ante/evm/03_global_fee.go#L20-L36)):

\`\`\`go func CheckGlobalFee(fee, globalMinGasPrice, gasLimit LegacyDec) error { if globalMinGasPrice.IsZero() { return nil }

```
  requiredFee := globalMinGasPrice.Mul(gasLimit)
  if fee.LT(requiredFee) {
      return ErrInsufficientFee
  }
  return nil
```

}

````

For EIP-1559 transactions, if the effective price falls below `MinGasPrice`, users must increase the priority fee (tip) until the effective price meets the minimum.

In Ontomir/evm's experimental mempool, transactions are ordered by effective gas tip (calculated as `min(maxPriorityFeePerGas, maxFeePerGas - baseFee)`), with higher tips getting priority. When tips are equal, transactions are ordered by arrival time ([source](https://github.com/Ontomir/evm/blob/v0.4.1/mempool/miner/ordering.go#L63-L71)).
</Expandable>

##### CheckGasWanted

Tracks cumulative gas wanted for base fee calculation ([source](https://github.com/Ontomir/evm/blob/v0.4.1/ante/evm/10_gas_wanted.go#L48-L82)):

<Expandable title="Implementation details">
```go
func CheckGasWanted(ctx Context, feeMarketKeeper FeeMarketKeeper, tx Tx, isLondon bool) error {
    if !isLondon || !feeMarketKeeper.GetBaseFeeEnabled(ctx) {
        return nil
    }

    gasWanted := tx.GetGas()
    blockGasLimit := BlockGasLimit(ctx)

    // Reject if tx exceeds block limit
    if gasWanted > blockGasLimit {
        return ErrOutOfGas
    }

    // Add to cumulative gas wanted for base fee adjustment
    feeMarketKeeper.AddTransientGasWanted(ctx, gasWanted)
    return nil
}
````

This cumulative gas wanted is used at the end of the block to calculate the next block's base fee.

### Module Distinctions <a href="#module-distinctions" id="module-distinctions"></a>

#### Ontomir/evm vs Standard Ontomir SDK <a href="#ontomirevm-vs-standard-ontomir-sdk" id="ontomirevm-vs-standard-ontomir-sdk"></a>

This fee market module from [Ontomir/evm](https://github.com/Ontomir/evm/tree/v0.4.1/x/feemarket) differs from standard Ontomir SDK fee handling:

| **Aspect**            | **Standard Ontomir SDK**                    | **Ontomir/evm Feemarket**                 |
| --------------------- | ------------------------------------------- | ----------------------------------------- |
| **Fee Model**         | Static, validator-configured min gas prices | Dynamic EIP-1559 base fee                 |
| **Price Discovery**   | Manual adjustment by validators             | Automatic based on utilization            |
| **Transaction Types** | Ontomir SDK Tx only                         | Both Ontomir & Ethereum txs               |
| **Decimal Handling**  | Chain-specific precision                    | 18-decimal EVM compatibility              |
| **State Storage**     | Not tracked                                 | Base fee in state, gas in transient store |
| **Consensus Impact**  | No consensus on fees                        | Base fee part of consensus                |

#### Key Integration Points <a href="#key-integration-points" id="key-integration-points"></a>

The module integrates with EVM through:

* **AnteHandlers**: Custom decorators for EVM transaction validation ([source](https://github.com/Ontomir/evm/blob/v0.4.1/ante/evm/))
* **Hooks**: PostTxProcessing for EVM state updates ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/feemarket/keeper/hooks.go))
* **RPC**: Ethereum JSON-RPC methods for fee queries ([source](https://github.com/Ontomir/evm/blob/v0.4.1/rpc/backend/chain_info.go))

### References <a href="#references" id="references"></a>

#### Source Code <a href="#source-code" id="source-code"></a>

* [Ontomir/evm Fee Market Module](https://github.com/Ontomir/evm/tree/v0.4.1/x/feemarket) - Main module implementation
* [AnteHandlers](https://github.com/Ontomir/evm/tree/v0.4.1/ante/evm) - EVM transaction validation
* [Proto Definitions](https://github.com/Ontomir/evm/tree/v0.4.1/proto/Ontomir/evm/feemarket/v1) - API and state structures
* [RPC Implementation](https://github.com/Ontomir/evm/tree/v0.4.1/rpc/backend) - Ethereum JSON-RPC support

#### Specifications <a href="#specifications" id="specifications"></a>

* [EIP-1559 Specification](https://eips.ethereum.org/EIPS/eip-1559) - Original Ethereum proposal
* [Ontomir SDK Docs](https://docs.ontomir.network/) - Base framework documentation

#### Tools & Libraries <a href="#tools-andamp-libraries" id="tools-andamp-libraries"></a>

* [Ethereum Gas Tracker](https://etherscan.io/gastracker) - Real-time Ethereum gas prices
* [Web3.js EIP-1559 Support](https://web3js.readthedocs.io/en/v1.7.0/web3-eth.html#id81) - JavaScript integration
* [Ethers.js Fee Data](https://docs.ethers.io/v5/api/providers/provider/#Provider-getFeeData) - TypeScript/JavaScript library
