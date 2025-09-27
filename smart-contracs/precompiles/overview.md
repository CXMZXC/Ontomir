# Overview

> Precompiles are predefined functions that are integrated at the protocol level but exposed as EVM smart contract interfaces. Many precompiles provide access to Ontomir SDK module functionality for EVM applications and clients to easily leverage.

### Available Precompiles <a href="#available-precompiles" id="available-precompiles"></a>

| Precompile       | Address                                      | Purpose                                                           | Reference |
| ---------------- | -------------------------------------------- | ----------------------------------------------------------------- | --------- |
| **Bank**         | `0x0000000000000000000000000000000000000804` | ERC20-style access to native Ontomir SDK tokens                   | Details   |
| **Bech32**       | `0x0000000000000000000000000000000000000400` | Address format conversion between Ethereum hex and Ontomir bech32 | Details   |
| **Staking**      | `0x0000000000000000000000000000000000000800` | Validator operations, delegation, and staking rewards             | Details   |
| **Distribution** | `0x0000000000000000000000000000000000000801` | Staking rewards and community pool management                     | Details   |
| **ERC20**        | Dynamic per token                            | Standard ERC20 functionality for native Ontomir tokens            | Details   |
| **Governance**   | `0x0000000000000000000000000000000000000805` | On-chain governance proposals and voting                          | Details   |
| **ICS20**        | `0x0000000000000000000000000000000000000802` | Cross-chain token transfers via IBC                               | Details   |
| **WERC20**       | Dynamic per token                            | Wrapped native token functionality                                | Details   |
| **Slashing**     | `0x0000000000000000000000000000000000000806` | Validator slashing and jail management                            | Details   |
| **P256**         | `0x0000000000000000000000000000000000000100` | P-256 elliptic curve cryptographic operations                     | Details   |

### Configuration <a href="#configuration" id="configuration"></a>

Precompiled contracts provide direct, gas-efficient access to native Ontomir SDK functionality from within the EVM. Build powerful hybrid applications that leverage the best of both worlds. This page provides an overview of the available precompiled contracts, each with detailed documentation on its usage, Solidity interface, and ABI.

\*\*Critical: Understanding Token Decimals in Precompiles\*\*

All Ontomir EVM precompile contracts use the **native chain's decimal precision**, not Ethereum's standard 18 decimals (wei).

**Why this matters:**

* Ontomir chains typically use 6 decimals, while Ethereum uses 18
* Passing values while assuming conventional Ethereum exponents could cause significant miscalculations
* Example: On a 6-decimal chain, passing `1000000000000000000` (1 token in wei) will be interpreted as **1 trillion tokens** **Before using any precompile:**

1. Check your chain's native token decimal precision
2. Convert amounts to match the native token's format
3.  Never assume 18-decimal precision

    ```
    // WRONG: Assuming Ethereum's 18 decimals
    uint256 amount = 1 ether; // 1000000000000000000
    staking.delegate(validator, amount); // Delegates 1 trillion tokens!

    // CORRECT: Using chain's native 6 decimals
    uint256 amount = 1000000; // 1 token with 6 decimals
    staking.delegate(validator, amount); // Delegates 1 token
    ```

#### Genesis Configuration <a href="#genesis-configuration" id="genesis-configuration"></a>

Precompiles must be enabled in the genesis file to be available on the network. The `active_static_precompiles` parameter controls which precompiles are activated at network launch.

```
{
  "app_state": {
    "evm": {
      "params": {
        "active_static_precompiles": [
          "0x0000000000000000000000000000000000000100",  // P256
          "0x0000000000000000000000000000000000000400",  // Bech32
          "0x0000000000000000000000000000000000000800",  // Staking
          "0x0000000000000000000000000000000000000801",  // Distribution
          "0x0000000000000000000000000000000000000802",  // ICS20
          "0x0000000000000000000000000000000000000804",  // Bank
          "0x0000000000000000000000000000000000000805",  // Governance
          "0x0000000000000000000000000000000000000806",  // Slashing
          "0x0000000000000000000000000000000000000807"   // Callbacks
        ]
      }
    }
  }
}
```

For complete genesis configuration including EVM and fee market parameters, see the \[Node Configuration]\(/docs/evm/next/documentation/getting-started/node-configuration#genesis-json) reference.

### v0.5.0 Breaking Changes <a href="#v050-breaking-changes" id="v050-breaking-changes"></a>

\*\*Precompile Constructor Interface Changes\*\*

All precompile constructors in v0.5.0 now require keeper interfaces instead of concrete keeper implementations. This improves decoupling and testability but requires updates to custom precompile implementations.

#### Interface Requirements <a href="#interface-requirements" id="interface-requirements"></a>

**Before (v0.4.x):**

```
// Concrete keeper types required
bankPrecompile, err := bankprecompile.NewPrecompile(
    bankKeeper,      // bankkeeper.Keeper concrete type
    stakingKeeper,   // stakingkeeper.Keeper concrete type
    // ... other keepers
)
```

**After (v0.5.0):**

```
// Keeper interfaces required (parameters accept interfaces directly)
bankPrecompile, err := bankprecompile.NewPrecompile(
    bankKeeper,    // common.BankKeeper interface
    erc20Keeper,   // erc20keeper.Keeper interface
)
```



#### Required Interfaces <a href="#required-interfaces" id="required-interfaces"></a>

The following keeper interfaces are defined in :

* `BankKeeper`: Account balances, token transfers, metadata
* `StakingKeeper`: Validator operations, delegations, bond denom
* `DistributionKeeper`: Reward withdrawals and distribution
* `SlashingKeeper`: Validator slashing information
* `TransferKeeper`: IBC transfer operations
* `ChannelKeeper`: IBC channel and connection queries
* `ERC20Keeper`: Token pair mappings and conversions

#### Migration Example <a href="#migration-example" id="migration-example"></a>

**Custom Precompile Migration:**

```
// Before (v0.4.x)
func NewCustomPrecompile(
    bankKeeper bankkeeper.Keeper,
    stakingKeeper stakingkeeper.Keeper,
) (*CustomPrecompile, error) {
    return &CustomPrecompile{
        bankKeeper:    bankKeeper,
        stakingKeeper: stakingKeeper,
    }, nil
}

// After (v0.5.0)
func NewCustomPrecompile(
    bankKeeper common.BankKeeper,        // Interface instead of concrete type
    stakingKeeper common.StakingKeeper,  // Interface instead of concrete type
) (*CustomPrecompile, error) {
    return &CustomPrecompile{
        bankKeeper:    bankKeeper,
        stakingKeeper: stakingKeeper,
    }, nil
}
```

**Assembly Updates:**

```
// In your precompile assembly function (based on evmd/precompiles.go)
func NewAvailableStaticPrecompiles(
    stakingKeeper stakingkeeper.Keeper,
    distributionKeeper distributionkeeper.Keeper,
    bankKeeper cmn.BankKeeper,              // Already interface type
    erc20Keeper erc20keeper.Keeper,
    // ... other keeper interfaces
    opts ...Option,
) map[common.Address]vm.PrecompiledContract {
    precompiles := make(map[common.Address]vm.PrecompiledContract)

    // No casting needed - parameters are already interface types
    bankPrecompile, err := bankprecompile.NewPrecompile(bankKeeper, erc20Keeper)
    if err != nil {
        panic(fmt.Errorf("failed to instantiate bank precompile: %w", err))
    }
    precompiles[bankPrecompile.Address()] = bankPrecompile
)
if err != nil {
    return nil, err
}
precompiles[bankAddress] = bankPrecompile
```

### Integration Guide <a href="#integration-guide" id="integration-guide"></a>

#### Activation <a href="#activation" id="activation"></a>

Precompiles must be explicitly activated via the `active_static_precompiles` parameter:

```
# Via genesis configuration
{
  "active_static_precompiles": [
    "0x0000000000000000000000000000000000000804", // Bank
    "0x0000000000000000000000000000000000000800"  // Staking
  ]
}

# Via governance proposal
evmd tx gov submit-proposal param-change-proposal.json --from validator
```

#### Usage in Smart Contracts <a href="#usage-in-smart-contracts" id="usage-in-smart-contracts"></a>

```
// Import precompile interfaces
import "./IBank.sol";
import "./IStaking.sol";

contract MyDApp {
    IBank constant bank = IBank(0x0000000000000000000000000000000000000804);
    IStaking constant staking = IStaking(0x0000000000000000000000000000000000000800);

    function getBalance(string memory denom) external view returns (uint256) {
        return bank.balances(msg.sender, denom);
    }

    function delegateTokens(string memory validatorAddr, uint256 amount) external {
        staking.delegate(msg.sender, validatorAddr, amount);
    }
}
```

#### Testing <a href="#testing" id="testing"></a>

All precompiles include comprehensive Solidity test suites for validation:

```
# Run precompile tests
cd tests && npm test

# Test specific precompile
npx hardhat test test/Bank.test.js
```
