# VM

> Ethereum Virtual Machine execution environment for Ontomir SDK

The `x/vm` module from Ontomir evm provides a fully compatible Ethereum Virtual Machine execution environment as a Ontomir SDK module.

For conceptual understanding of EVM architecture and design, see \[EVM Architecture]\(/docs/evm/next/documentation/concepts/overview).

### Parameters <a href="#parameters" id="parameters"></a>

The module parameters control EVM behavior and access policies :

| Parameter                   | Type          | Default        | Description                                           |
| --------------------------- | ------------- | -------------- | ----------------------------------------------------- |
| `evm_denom`                 | string        | `"atest"`      | Token denomination for EVM transactions (18 decimals) |
| `extra_eips`                | \[]int64      | `[]`           | Additional EIPs to activate beyond hard fork defaults |
| `active_static_precompiles` | \[]string     | `[]`           | Activated static precompile addresses                 |
| `evm_channels`              | \[]string     | `[]`           | IBC channel IDs for EVM-compatible chains             |
| `access_control`            | AccessControl | Permissionless | Permission policy for EVM operations                  |
| `history_serve_window`      | uint64        | `8192`         | Historical block hash depth for EIP-2935 (v0.5.0+)    |

\*\*Breaking Change in v0.5.0:\*\* The \`allow\_unprotected\_txs\` parameter has been removed from consensus parameters. Non-EIP155 transaction acceptance is now configured per-node in \`app.toml\` under \`\[json-rpc].allow-unprotected-txs\`.

#### Parameter Details <a href="#parameter-details" id="parameter-details"></a>

Defines the token used for gas and value transfers in EVM:

* Must be registered in bank module
* Should have 18 decimal precision
* Used for all EVM operations (gas, transfers, contracts)
* Example: `"atest"`, `"aevmos"`, `"wei"`

Activates additional EIPs beyond fork defaults:

| EIP  | Description                                 | Gas Impact            |
| ---- | ------------------------------------------- | --------------------- |
| 1344 | ChainID opcode                              | Minimal               |
| 1884 | Repricing for trie-size-dependent opcodes   | Increases SLOAD       |
| 2200 | Structured definitions for net gas metering | Variable              |
| 2929 | Gas cost for state access opcodes           | Increases cold access |
| 3198 | BASEFEE opcode                              | Minimal               |
| 3529 | Reduction in refunds                        | Reduces max refund    |
| 3855 | PUSH0 instruction                           | New opcode            |
|      |                                             |                       |

Controls EIP-2935 historical block hash storage depth:

**Purpose:** Enables smart contracts to access historical block hashes via `BLOCKHASH` opcode **Range:** Must be > 0, recommended ≤ 8192 for optimal performance **Storage Impact:** Higher values increase node storage requirements

**Configuration Examples:**

```
{
  "history_serve_window": 8192   // Default: full EIP-2935 compatibility
}
```

```
{
  "history_serve_window": 1024   // Performance optimized: reduced storage
}
```

```
{
  "history_serve_window": 16384  // Extended history: maximum compatibility
}
```

**Smart Contract Usage:**

```
contract HistoryExample {
    function getRecentBlockHash(uint256 blockNumber) public view returns (bytes32) {
        // Works for blocks within history_serve_window range
        return blockhash(blockNumber);
    }
}
```

Granular control over contract operations:

```
type AccessControl struct {
    Create AccessControlType  // Contract deployment
    Call   AccessControlType  // Contract execution
}
```

**Access Types:**

* `PERMISSIONLESS`: Open access, list = blacklist
* `RESTRICTED`: No access, list ignored
* `PERMISSIONED`: Limited access, list = whitelist

### Configuration Profiles <a href="#configuration-profiles" id="configuration-profiles"></a>

\`\`\`json { "evm\_denom": "atest", "extra\_eips": \[3855], "active\_static\_precompiles": \[ "0x0000000000000000000000000000000000000100", "0x0000000000000000000000000000000000000400" ], "evm\_channels": \[], "access\_control": { "create": {"access\_type": "PERMISSIONLESS", "access\_control\_list": \[]}, "call": {"access\_type": "PERMISSIONLESS", "access\_control\_list": \[]} }, "history\_serve\_window": 8192 } \`\`\` \`\`\`json { "evm\_denom": "atoken", "extra\_eips": \[], "active\_static\_precompiles": \[ "0x0000000000000000000000000000000000000804" ], "evm\_channels": \[], "access\_control": { "create": { "access\_type": "PERMISSIONED", "access\_control\_list": \["0x1234...", "0xABCD..."] }, "call": {"access\_type": "PERMISSIONLESS", "access\_control\_list": \[]} }, "history\_serve\_window": 4096 } \`\`\` \`\`\`json { "evm\_denom": "asecure", "extra\_eips": \[], "active\_static\_precompiles": \[], "evm\_channels": \[], "access\_control": { "create": {"access\_type": "RESTRICTED", "access\_control\_list": \[]}, "call": { "access\_type": "PERMISSIONED", "access\_control\_list": \["0xTrustedContract"] } }, "history\_serve\_window": 1024 } \`\`\`

### Chain Configuration <a href="#chain-configuration" id="chain-configuration"></a>

Hard fork activation schedule :

| Fork                   | Default | Type      | Opcodes/Features Added               |
| ---------------------- | ------- | --------- | ------------------------------------ |
| `homestead_block`      | 0       | Block     | DELEGATECALL                         |
| `byzantium_block`      | 0       | Block     | REVERT, RETURNDATASIZE, STATICCALL   |
| `constantinople_block` | 0       | Block     | CREATE2, EXTCODEHASH, SHL/SHR/SAR    |
| `istanbul_block`       | 0       | Block     | CHAINID, SELFBALANCE                 |
| `berlin_block`         | 0       | Block     | Access lists (EIP-2929, 2930)        |
| `london_block`         | 0       | Block     | BASEFEE, EIP-1559                    |
| `shanghai_time`        | 0       | Timestamp | PUSH0                                |
| `cancun_time`          | 0       | Timestamp | Transient storage, blob transactions |
| `prague_time`          | nil     | Timestamp | TBD                                  |
| `verkle_time`          | nil     | Timestamp | Verkle trees                         |

### Static Precompiles <a href="#static-precompiles" id="static-precompiles"></a>

Precompiled contracts at fixed addresses :

| Address         | Contract         | Gas Cost | Description             |
| --------------- | ---------------- | -------- | ----------------------- |
| `0x0000...0100` | **P256**         | 3000     | P256 curve verification |
| `0x0000...0400` | **Bech32**       | 5000     | Bech32 address encoding |
| `0x0000...0800` | **Staking**      | Variable | Delegation operations   |
| `0x0000...0801` | **Distribution** | Variable | Reward claims           |
| `0x0000...0802` | **ICS20**        | Variable | IBC transfers           |
| `0x0000...0803` | **Vesting**      | Variable | Vesting account ops     |
| `0x0000...0804` | **Bank**         | Variable | Token transfers         |
| `0x0000...0805` | **Gov**          | Variable | Governance voting       |
| `0x0000...0806` | **Slashing**     | Variable | Validator slashing info |

#### Activation <a href="#activation" id="activation"></a>

```
# Via genesis
{
  "active_static_precompiles": [
    "0x0000000000000000000000000000000000000804",
    "0x0000000000000000000000000000000000000800"
  ]
}

# Via governance proposal
evmd tx gov submit-proposal update-params-proposal.json
```

### Messages <a href="#messages" id="messages"></a>

#### MsgEthereumTx <a href="#msgethereumtx" id="msgethereumtx"></a>

Primary message for EVM transactions :

```
message MsgEthereumTx {
  google.protobuf.Any data = 1;  // Transaction data (Legacy/AccessList/DynamicFee)
  string hash = 3;                // Transaction hash
  string from = 4;                // Sender address (derived from signature)
}
```

**Validation Requirements:**

* Valid signature matching `from` address
* Sufficient balance for gas + value
* Correct nonce (account sequence)
* Gas limit ≥ intrinsic gas

#### MsgUpdateParams <a href="#msgupdateparams" id="msgupdateparams"></a>

Governance message for parameter updates:

```
message MsgUpdateParams {
  string authority = 1;  // Must be governance module
  Params params = 2;     // New parameters
}
```

### State <a href="#state" id="state"></a>

#### Persistent State <a href="#persistent-state" id="persistent-state"></a>

| Key Prefix | Description      | Value Type |
| ---------- | ---------------- | ---------- |
| `0x01`     | Parameters       | `Params`   |
| `0x02`     | Contract code    | `[]byte`   |
| `0x03`     | Contract storage | `[32]byte` |

#### Transient State <a href="#transient-state" id="transient-state"></a>

| Key Prefix | Description      | Value Type  | Lifetime  |
| ---------- | ---------------- | ----------- | --------- |
| `0x01`     | Transaction logs | `[]Log`     | Per block |
| `0x02`     | Block bloom      | `[256]byte` | Per block |
| `0x03`     | Tx index         | `uint64`    | Per block |
| `0x04`     | Log size         | `uint64`    | Per block |
| `0x05`     | Gas used         | `uint64`    | Per tx    |

### Events <a href="#events" id="events"></a>

#### Transaction Events <a href="#transaction-events" id="transaction-events"></a>

| Event         | Attributes                                                          | When Emitted               |
| ------------- | ------------------------------------------------------------------- | -------------------------- |
| `ethereum_tx` | `amount`, `recipient`, `contract`, `txHash`, `txIndex`, `txGasUsed` | Every EVM transaction      |
| `tx_log`      | `txLog` (JSON)                                                      | When contracts emit events |
| `message`     | `sender`, `action="ethereum"`, `module="evm"`                       | Every message              |

#### Block Events <a href="#block-events" id="block-events"></a>

| Event         | Attributes    | When Emitted                   |
| ------------- | ------------- | ------------------------------ |
| `block_bloom` | `bloom` (hex) | End of each block with EVM txs |

### Queries <a href="#queries" id="queries"></a>

#### gRPC <a href="#grpc" id="grpc"></a>

\`\`\`protobuf service Query { // Account queries rpc Account(QueryAccountRequest) returns (QueryAccountResponse); rpc OntomirAccount(QueryOntomirAccountRequest) returns (QueryOntomirAccountResponse); rpc ValidatorAccount(QueryValidatorAccountRequest) returns (QueryValidatorAccountResponse);

```
// State queries
rpc Balance(QueryBalanceRequest) returns (QueryBalanceResponse);
rpc Storage(QueryStorageRequest) returns (QueryStorageResponse);
rpc Code(QueryCodeRequest) returns (QueryCodeResponse);

// Module queries
rpc Params(QueryParamsRequest) returns (QueryParamsResponse);
rpc Config(QueryConfigRequest) returns (QueryConfigResponse);

// EVM operations
rpc EthCall(EthCallRequest) returns (MsgEthereumTxResponse);
rpc EstimateGas(EthCallRequest) returns (EstimateGasResponse);

// Debugging
rpc TraceTx(QueryTraceTxRequest) returns (QueryTraceTxResponse);
rpc TraceBlock(QueryTraceBlockRequest) returns (QueryTraceBlockResponse);

// Fee queries
rpc BaseFee(QueryBaseFeeRequest) returns (QueryBaseFeeResponse);
rpc GlobalMinGasPrice(QueryGlobalMinGasPriceRequest) returns (QueryGlobalMinGasPriceResponse);
```

}

````
</Expandable>

### CLI

<CodeGroup>
```bash Query-Account
# Query EVM account
evmd query vm account 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb0

# Query Ontomir address
evmd query vm Ontomir-account 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb0
````

```
# Query balance
evmd query vm balance 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb0

# Query storage slot
evmd query vm storage 0xContractAddress 0xStorageKey

# Query contract code
evmd query vm code 0xContractAddress
```

```
# Query module parameters
evmd query vm params

# Query EVM config
evmd query vm config
```

```
# Trace transaction
evmd query vm trace-tx 0xTxHash

# Trace block
evmd query vm trace-block 12345
```

### JSON-RPC <a href="#json-rpc" id="json-rpc"></a>

#### Supported Methods <a href="#supported-methods" id="supported-methods"></a>

\| Method | Description | | --------------------------- | -------------------------------------------------------- | | \`eth\_accounts\` | List accounts | | \`eth\_blockNumber\` | Current block number | | \`eth\_call\` | Execute call without tx | | \`eth\_chainId\` | Get chain ID | | \`eth\_createAccessList\` | Generate access list for tx optimization (v0.5.0+) | | \`eth\_estimateGas\` | Estimate gas usage (optimized in v0.5.0+) | | \`eth\_gasPrice\` | Current gas price | | \`eth\_getBalance\` | Get account balance | | \`eth\_getBlockByHash\` | Get block by hash | | \`eth\_getBlockByNumber\` | Get block by number | | \`eth\_getCode\` | Get contract code | | \`eth\_getLogs\` | Get filtered logs | | \`eth\_getStorageAt\` | Get storage value | | \`eth\_getTransactionByHash\` | Get tx by hash | | \`eth\_getTransactionCount\` | Get account nonce | | \`eth\_getTransactionReceipt\` | Get tx receipt (enhanced with max\\\_used\\\_gas in v0.5.0+) | | \`eth\_sendRawTransaction\` | Submit signed tx | | \`eth\_syncing\` | Sync status | | Method | Description | | -------------------- | --------------- | | \`web3\_clientVersion\` | Client version | | \`web3\_sha3\` | Keccak-256 hash | | Method | Description | | --------------- | -------------------------- | | \`net\_version\` | Network ID | | \`net\_listening\` | Node accepting connections | | \`net\_peerCount\` | Number of peers | | Method | Description | | -------------------------- | ---------------------------- | | \`debug\_traceTransaction\` | Detailed tx trace | | \`debug\_traceBlockByNumber\` | Trace all txs in block | | \`debug\_traceBlockByHash\` | Trace all txs by hash | | \`debug\_getBadBlocks\` | Get invalid blocks | | \`debug\_storageRangeAt\` | Get storage range | | \`debug\_intermediateRoots\` | Get intermediate state roots |

### Hooks <a href="#hooks" id="hooks"></a>

Post-transaction processing interface :

```
type EvmHooks interface {
    PostTxProcessing(
        ctx sdk.Context,
        msg core.Message,
        receipt *ethtypes.Receipt,
    ) error
}
```

#### Registration <a href="#registration" id="registration"></a>

```
// In app.go
app.EvmKeeper = app.EvmKeeper.SetHooks(
    vmkeeper.NewMultiEvmHooks(
        app.Erc20Keeper.Hooks(),
        app.CustomKeeper.Hooks(),
    ),
)
```

#### Use Cases <a href="#use-cases" id="use-cases"></a>

* Process EVM events in Ontomir modules
* Bridge EVM logs to Ontomir events
* Trigger native module actions from contracts
* Custom indexing and monitoring

### Gas Configuration <a href="#gas-configuration" id="gas-configuration"></a>

#### Gas Consumption Mapping <a href="#gas-consumption-mapping" id="gas-consumption-mapping"></a>

| Operation  | EVM Gas  | Ontomir Gas | Notes              |
| ---------- | -------- | ----------- | ------------------ |
| Basic ops  | 3-5      | 3-5         | 1:1 mapping        |
| SLOAD      | 2100     | 2100        | Cold access        |
| SSTORE     | 20000    | 20000       | New value          |
| CREATE     | 32000    | 32000       | Contract deploy    |
| Precompile | Variable | Variable    | Set per precompile |

#### Gas Refunds <a href="#gas-refunds" id="gas-refunds"></a>

Maximum refund: 50% of gas used (after London)

```
refund = min(gasUsed/2, gasRefund)
finalGasUsed = gasUsed - refund
```

### Best Practices <a href="#best-practices" id="best-practices"></a>

#### Chain Integration <a href="#chain-integration" id="chain-integration"></a>

1. **Precompile Selection**
   * Only enable necessary precompiles
   * Test gas costs in development
   * Monitor usage patterns
2. **Access Control**
   * Start restrictive, open gradually
   * Use governance for changes
   * Monitor blacklist/whitelist events
3.  **Fork Planning**

    ```
    {
      "shanghai_time": "1681338455",  // Test first
      "cancun_time": "1710338455"     // Delay for safety
    }
    ```

#### Contract Development <a href="#contract-development" id="contract-development"></a>

1.  **Gas Optimization**

    ```
    // Cache storage reads
    uint256 cached = storageVar;
    // Use cached instead of storageVar

    // Pack structs
    struct Packed {
        uint128 a;
        uint128 b;  // Single slot
    }
    ```
2.  **Precompile Usage**

    ```
    IBank bank = IBank(0x0000000000000000000000000000000000000804);
    uint256 balance = bank.balances(msg.sender, "atest");
    ```

#### dApp Integration <a href="#dapp-integration" id="dapp-integration"></a>

1.  **Provider Setup**

    ```
    const provider = new ethers.JsonRpcProvider(
        "http://localhost:8545",
        { chainId: 9000, name: "evmos" }
    );
    ```
2.  **Error Handling**

    ```
    try {
        await contract.method();
    } catch (error) {
        if (error.code === 'CALL_EXCEPTION') {
            // Handle revert
        }
    }
    ```

### Troubleshooting <a href="#troubleshooting" id="troubleshooting"></a>

#### Common Issues <a href="#common-issues" id="common-issues"></a>

| Issue                      | Cause                         | Solution                                       |
| -------------------------- | ----------------------------- | ---------------------------------------------- |
| "out of gas"               | Insufficient gas limit        | Increase gas limit or optimize contract        |
| "nonce too low"            | Stale nonce                   | Query current nonce, retry                     |
| "insufficient funds"       | Low balance                   | Ensure balance > value + (gasLimit × gasPrice) |
| "contract creation failed" | CREATE disabled or restricted | Check access\_control settings                 |
| "precompile not active"    | Precompile not in active list | Enable via governance                          |

#### Debug Commands <a href="#debug-commands" id="debug-commands"></a>

```
# Check EVM parameters
evmd query vm params

# Trace failed transaction
evmd query vm trace-tx 0xFailedTxHash

# Check account state
evmd query vm account 0xAddress
```

### References <a href="#references" id="references"></a>

#### Source Code <a href="#source-code" id="source-code"></a>

* EVM Architecture - Conceptual overview
* Fee Market Module - EIP-1559 implementation
* ERC20 Module - Token conversions
