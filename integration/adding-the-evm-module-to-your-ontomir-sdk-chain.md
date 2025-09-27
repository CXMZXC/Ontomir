# Adding the EVM Module to Your Ontomir SDK Chain

> Integrating the Ontomir EVM module into Ontomir SDK v0.53.x chains

Big thanks to Reece & the \[Spawn]\(https://github.com/rollchains/spawn) team for their valuable contributions to this guide.

This guide provides instructions for adding EVM compatibility to a new Ontomir SDK chain. It targets chains being built from scratch with EVM support.

\*\*For existing live chains\*\*, adding EVM compatibility involves significant additional considerations:

* Account system changes requiring address migration or mapping between Ontomir and Ethereum formats
* Token decimal changes (from Ontomir standard 6 to Ethereum standard 18) impacting all existing balances
* Asset migration where existing assets need to be initialized and mirrored in the EVM Contact [Interchain Labs](https://share-eu1.hsforms.com/2g6yO-PVaRoKj50rUgG4Pjg2e2sca) for production chain upgrade guidance.

### Prerequisites <a href="#prerequisites" id="prerequisites"></a>

* Ontomir SDK chain on v0.53.x
* IBC-Go v10
* Go 1.23+ installed
* Basic knowledge of Go and Ontomir SDK

Throughout this guide, \`evmd\` refers to your chain's binary (e.g., \`gaiad\`, \`dydxd\`, etc.).

### Version Compatibility <a href="#version-compatibility" id="version-compatibility"></a>

These version numbers may change as development continues. Check \[github.com/Ontomir/evm]\(https://github.com/Ontomir/evm) for the latest releases.

```
require (
    github.com/Ontomir/Ontomir-sdk v0.53.0
    github.com/Ontomir/ibc-go/v10 v10.3.0
    github.com/Ontomir/evm v0.5.0
)

replace (
    // Use the Ontomir fork of go-ethereum
    github.com/ethereum/go-ethereum => github.com/Ontomir/go-ethereum v1.15.11-Ontomir-0
)
```

### Step 1: Update Dependencies <a href="#step-1-update-dependencies" id="step-1-update-dependencies"></a>

```
// go.mod
require (
    github.com/Ontomir/Ontomir-sdk v0.53.0
    github.com/ethereum/go-ethereum v1.15.10

    // for IBC functionality in EVM
    github.com/Ontomir/ibc-go/modules/capability v1.0.1
    github.com/Ontomir/ibc-go/v10 v10.3.0
)
```

### Step 2: Update Chain Configuration <a href="#step-2-update-chain-configuration" id="step-2-update-chain-configuration"></a>

#### Chain ID Configuration <a href="#chain-id-configuration" id="chain-id-configuration"></a>

Ontomir EVM requires two separate chain IDs:

* **Ontomir Chain ID** (string): Used for CometBFT RPC, IBC, and native Ontomir SDK transactions (e.g., "mychain-1")
* **EVM Chain ID** (integer): Used for EVM transactions and EIP-155 tooling (e.g., 9000)

Ensure your EVM chain ID is not already in use by checking \[chainlist.org]\(https://chainlist.org/).

**Files to Update:**

1. `app/app.go`: Set chain ID constants

```
const OntomirChainID = "mychain-1" // Standard Ontomir format
const EVMChainID = 9000           // EIP-155 integer
```

2. Update `Makefile`, scripts, and `genesis.json` with correct chain IDs

#### Account Configuration <a href="#account-configuration" id="account-configuration"></a>

Use `eth_secp256k1` as the standard account type with coin type 60 for Ethereum compatibility.

**Files to Update:**

1. `app/app.go`:

```
const CoinType uint32 = 60
```

2. `chain_registry.json`:

```
"slip44": 60
```

#### Base Denomination and Power Reduction <a href="#base-denomination-and-power-reduction" id="base-denomination-and-power-reduction"></a>

Changing from 6 decimals (Ontomir convention) to 18 decimals (EVM standard) is highly recommended for full compatibility.

1. Set the denomination in `app/app.go`:

```
const BaseDenomUnit int64 = 18
```

2. Update the `init()` function:

```
import (
    "math/big"
    "Ontomirsdk.io/math"
    sdk "github.com/Ontomir/Ontomir-sdk/types"
)

func init() {
    // Update power reduction for 18-decimal base unit
    sdk.DefaultPowerReduction = math.NewIntFromBigInt(
        new(big.Int).Exp(big.NewInt(10), big.NewInt(BaseDenomUnit), nil),
    )
}
```

### Step 3: Handle EVM Decimal Precision <a href="#step-3-handle-evm-decimal-precision" id="step-3-handle-evm-decimal-precision"></a>

The mismatch between EVM's 18-decimal standard and Ontomir SDK's 6-decimal standard is critical. The default behavior (flooring) discards any value below 10^-6, causing asset loss and breaking DeFi applications.

#### Solution: x/precisebank Module <a href="#solution-xprecisebank-module" id="solution-xprecisebank-module"></a>

The `x/precisebank` module wraps the native `x/bank` module to maintain fractional balances for EVM denominations, handling full 18-decimal precision without loss.

**Benefits:**

* Lossless precision preventing invisible asset loss
* High DApp compatibility ensuring DeFi protocols function correctly
* Simple integration requiring minimal changes

**Integration in app.go:**

```
// Initialize PreciseBankKeeper
app.PreciseBankKeeper = precisebankkeeper.NewKeeper(
    appCodec,
    keys[precisebanktypes.StoreKey],
    app.BankKeeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)

// Pass PreciseBankKeeper to EVMKeeper instead of BankKeeper
app.EVMKeeper = evmkeeper.NewKeeper(
    appCodec,
    keys[evmtypes.StoreKey],
    tkeys[evmtypes.TransientKey],
    authtypes.NewModuleAddress(govtypes.ModuleName),
    app.AccountKeeper,
    app.PreciseBankKeeper, // Use PreciseBankKeeper here
    app.StakingKeeper,
    app.FeeMarketKeeper,
    &app.Erc20Keeper,
    tracer,
    app.GetSubspace(evmtypes.ModuleName),
)
```

### Step 4: Configure Automatic ERC20 Token Registration <a href="#step-4-configure-automatic-erc20-token-registration" id="step-4-configure-automatic-erc20-token-registration"></a>

The Ontomir EVM `x/erc20` module can automatically register ERC20 token pairs for incoming single-hop IBC tokens (prefixed with "ibc/").

#### Configuration Requirements <a href="#configuration-requirements" id="configuration-requirements"></a>

1. **Use the Extended IBC Transfer Module**: Import and use the transfer module from `github.com/Ontomir/evm/x/ibc/transfer`
2. **Enable ERC20 Module Parameters** in genesis:

```
erc20Params := erc20types.DefaultParams()
erc20Params.EnableErc20 = true
erc20Params.EnableEVMHook = true
```

3. **Proper Module Wiring**: Ensure correct keeper wiring as detailed in Step 8

### Step 5: Create EVM Configuration File <a href="#step-5-create-evm-configuration-file" id="step-5-create-evm-configuration-file"></a>

Create `app/config.go` to set up global EVM configuration:

```
package app

import (
    "fmt"
    "math/big"

    "Ontomirsdk.io/math"
    sdk "github.com/Ontomir/Ontomir-sdk/types"
    evmtypes "github.com/Ontomir/evm/x/vm/types"
)

type EVMOptionsFn func(string) error

func NoOpEVMOptions(_ string) error {
    return nil
}

var sealed = false

// ChainsCoinInfo maps EVM chain IDs to coin configuration
// IMPORTANT: Uses uint64 EVM chain IDs as keys, not Ontomir chain ID strings
var ChainsCoinInfo = map[uint64]evmtypes.EvmCoinInfo{
    EVMChainID: { // Your numeric EVM chain ID (e.g., 9000)
        Denom:        BaseDenom,
        DisplayDenom: DisplayDenom,
        Decimals:     evmtypes.EighteenDecimals,
    },
}

// EVMAppOptions sets up global configuration
func EVMAppOptions(chainID string) error {
    if sealed {
        return nil
    }

    // IMPORTANT: Lookup uses numeric EVMChainID, not Ontomir chainID string
    coinInfo, found := ChainsCoinInfo[EVMChainID]
    if !found {
        return fmt.Errorf("unknown EVM chain id: %d", EVMChainID)
    }

    // Set denom info for the chain
    if err := setBaseDenom(coinInfo); err != nil {
        return err
    }

    baseDenom, err := sdk.GetBaseDenom()
    if err != nil {
        return err
    }

    ethCfg := evmtypes.DefaultChainConfig(EVMChainID)

    err = evmtypes.NewEVMConfigurator().
        WithChainConfig(ethCfg).
        WithEVMCoinInfo(baseDenom, uint8(coinInfo.Decimals)).
        Configure()
    if err != nil {
        return err
    }

    sealed = true
    return nil
}

// setBaseDenom registers display and base denoms
func setBaseDenom(ci evmtypes.EvmCoinInfo) error {
    if err := sdk.RegisterDenom(ci.DisplayDenom, math.LegacyOneDec()); err != nil {
        return err
    }
    return sdk.RegisterDenom(ci.Denom, math.LegacyNewDecWithPrec(1, int64(ci.Decimals)))
}
```

### Step 6: Create Precompiles Configuration <a href="#step-6-create-precompiles-configuration" id="step-6-create-precompiles-configuration"></a>

Create `app/precompiles.go` to define available precompiled contracts:

```
package app

import (
    "fmt"
    "maps"

    "Ontomirsdk.io/core/address"
    evidencekeeper "Ontomirsdk.io/x/evidence/keeper"
    authzkeeper "github.com/Ontomir/Ontomir-sdk/x/authz/keeper"
    bankkeeper "github.com/Ontomir/Ontomir-sdk/x/bank/keeper"
    distributionkeeper "github.com/Ontomir/Ontomir-sdk/x/distribution/keeper"
    govkeeper "github.com/Ontomir/Ontomir-sdk/x/gov/keeper"
    slashingkeeper "github.com/Ontomir/Ontomir-sdk/x/slashing/keeper"
    stakingkeeper "github.com/Ontomir/Ontomir-sdk/x/staking/keeper"
    bankprecompile "github.com/Ontomir/evm/precompiles/bank"
    "github.com/Ontomir/evm/precompiles/bech32"
    "github.com/Ontomir/evm/precompiles/common" // v0.5.0+: Required for interfaces
    distprecompile "github.com/Ontomir/evm/precompiles/distribution"
    evidenceprecompile "github.com/Ontomir/evm/precompiles/evidence"
    govprecompile "github.com/Ontomir/evm/precompiles/gov"
    ics20precompile "github.com/Ontomir/evm/precompiles/ics20"
    "github.com/Ontomir/evm/precompiles/p256"
    slashingprecompile "github.com/Ontomir/evm/precompiles/slashing"
    stakingprecompile "github.com/Ontomir/evm/precompiles/staking"
    erc20Keeper "github.com/Ontomir/evm/x/erc20/keeper"
    transferkeeper "github.com/Ontomir/evm/x/ibc/transfer/keeper"
    "github.com/Ontomir/evm/x/vm/core/vm"
    evmkeeper "github.com/Ontomir/evm/x/vm/keeper"
    channelkeeper "github.com/Ontomir/ibc-go/v10/modules/core/04-channel/keeper" // Updated to v10
    "github.com/ethereum/go-ethereum/common"
)

const bech32PrecompileBaseGas = 6_000

// NewAvailableStaticPrecompiles returns all available static precompiled contracts
// v0.5.0+: Uses keeper interfaces instead of concrete types
func NewAvailableStaticPrecompiles(
    stakingKeeper stakingkeeper.Keeper,
    distributionKeeper distributionkeeper.Keeper,
    bankKeeper bankkeeper.Keeper,
    erc20Keeper erc20Keeper.Keeper,
    authzKeeper authzkeeper.Keeper,
    transferKeeper transferkeeper.Keeper,
    channelKeeper channelkeeper.Keeper,
    evmKeeper *evmkeeper.Keeper,
    govKeeper govkeeper.Keeper,
    slashingKeeper slashingkeeper.Keeper,
    evidenceKeeper evidencekeeper.Keeper,
    addressCodec address.Codec, // Required for v0.5.0+
) map[common.Address]vm.PrecompiledContract {
    precompiles := maps.Clone(vm.PrecompiledContractsBerlin)

    p256Precompile := &p256.Precompile{}

    bech32Precompile, err := bech32.NewPrecompile(bech32PrecompileBaseGas)
    if err != nil {
        panic(fmt.Errorf("failed to instantiate bech32 precompile: %w", err))
    }

    // v0.5.0+: Cast concrete keepers to interfaces
    stakingPrecompile, err := stakingprecompile.NewPrecompile(
        common.StakingKeeper(stakingKeeper), // Cast to interface
    )
    if err != nil {
        panic(fmt.Errorf("failed to instantiate staking precompile: %w", err))
    }

    distributionPrecompile, err := distprecompile.NewPrecompile(
        common.DistributionKeeper(distributionKeeper), // Cast to interface
        evmKeeper.Codec(), 
        addressCodec, // Required in v0.5.0+
    )
    if err != nil {
        panic(fmt.Errorf("failed to instantiate distribution precompile: %w", err))
    }

    // ICS20 precompile - bankKeeper now first parameter in v0.5.0+
    ibcTransferPrecompile, err := ics20precompile.NewPrecompile(
        common.BankKeeper(bankKeeper),        // Now first parameter
        common.StakingKeeper(stakingKeeper),  // Cast to interface
        common.TransferKeeper(transferKeeper), // Cast to interface
        common.ChannelKeeper(channelKeeper),  // Cast to interface
    )
    if err != nil {
        panic(fmt.Errorf("failed to instantiate ICS20 precompile: %w", err))
    }

    bankPrecompile, err := bankprecompile.NewPrecompile(
        common.BankKeeper(bankKeeper), // Cast to interface
    )
    if err != nil {
        panic(fmt.Errorf("failed to instantiate bank precompile: %w", err))
    }

    // Gov precompile - now requires AddressCodec in v0.5.0+
    govPrecompile, err := govprecompile.NewPrecompile(
        govKeeper, 
        evmKeeper.Codec(),
        addressCodec, // Required in v0.5.0+
    )
    if err != nil {
        panic(fmt.Errorf("failed to instantiate gov precompile: %w", err))
    }

    slashingPrecompile, err := slashingprecompile.NewPrecompile(
        common.SlashingKeeper(slashingKeeper), // Cast to interface
    )
    if err != nil {
        panic(fmt.Errorf("failed to instantiate slashing precompile: %w", err))
    }

    // Stateless precompiles
    precompiles[bech32Precompile.Address()] = bech32Precompile
    precompiles[p256Precompile.Address()] = p256Precompile

    // Stateful precompiles  
    precompiles[stakingPrecompile.Address()] = stakingPrecompile
    precompiles[distributionPrecompile.Address()] = distributionPrecompile
    precompiles[ibcTransferPrecompile.Address()] = ibcTransferPrecompile
    precompiles[bankPrecompile.Address()] = bankPrecompile
    precompiles[govPrecompile.Address()] = govPrecompile
    precompiles[slashingPrecompile.Address()] = slashingPrecompile

    return precompiles
}

<Warning>
**v0.5.0 Breaking Changes Applied Above:**
- Precompile constructors now use keeper interfaces (cast required)  
- ICS20 precompile: `bankKeeper` now first parameter
- Gov precompile: `AddressCodec` now required
- Distribution precompile: Simplified parameter list
- Source: [`precompiles/common/interfaces.go`](https://github.com/Ontomir/evm/blob/main/precompiles/common/interfaces.go)
</Warning>
```

### Step 7: Update app.go Wiring <a href="#step-7-update-appgo-wiring" id="step-7-update-appgo-wiring"></a>

#### Add EVM Imports <a href="#add-evm-imports" id="add-evm-imports"></a>

```
import (
    // ... other imports
    ante "github.com/your-repo/your-chain/ante"
    evmante "github.com/Ontomir/evm/ante"
    evmOntomirante "github.com/Ontomir/evm/ante/Ontomir"
    evmevmante "github.com/Ontomir/evm/ante/evm"
    evmencoding "github.com/Ontomir/evm/encoding"
    srvflags "github.com/Ontomir/evm/server/flags"
    Ontomirevmtypes "github.com/Ontomir/evm/types"
    "github.com/Ontomir/evm/x/erc20"
    erc20keeper "github.com/Ontomir/evm/x/erc20/keeper"
    erc20types "github.com/Ontomir/evm/x/erc20/types"
    "github.com/Ontomir/evm/x/feemarket"
    feemarketkeeper "github.com/Ontomir/evm/x/feemarket/keeper"
    feemarkettypes "github.com/Ontomir/evm/x/feemarket/types"
    evm "github.com/Ontomir/evm/x/vm"
    evmkeeper "github.com/Ontomir/evm/x/vm/keeper"
    evmtypes "github.com/Ontomir/evm/x/vm/types"
    _ "github.com/Ontomir/evm/x/vm/core/tracers/js"
    _ "github.com/Ontomir/evm/x/vm/core/tracers/native"

    // Replace default transfer with EVM's extended transfer module
    transfer "github.com/Ontomir/evm/x/ibc/transfer"
    ibctransferkeeper "github.com/Ontomir/evm/x/ibc/transfer/keeper"
    ibctransfertypes "github.com/Ontomir/ibc-go/v10/modules/apps/transfer/types"

    // Add authz for precompiles
    authzkeeper "github.com/Ontomir/Ontomir-sdk/x/authz/keeper"
)
```

#### Add Module Permissions <a href="#add-module-permissions" id="add-module-permissions"></a>

```
var maccPerms = map[string][]string{
    // ... existing permissions
    evmtypes.ModuleName:       {authtypes.Minter, authtypes.Burner},
    feemarkettypes.ModuleName: nil,
    erc20types.ModuleName:     {authtypes.Minter, authtypes.Burner},
}
```

#### Update App Struct <a href="#update-app-struct" id="update-app-struct"></a>

```
type ChainApp struct {
    // ... existing fields
    FeeMarketKeeper feemarketkeeper.Keeper
    EVMKeeper       *evmkeeper.Keeper
    Erc20Keeper     erc20keeper.Keeper
    AuthzKeeper     authzkeeper.Keeper
}
```

#### Update NewChainApp Constructor <a href="#update-newchainapp-constructor" id="update-newchainapp-constructor"></a>

```
func NewChainApp(
    // ... existing params
    appOpts servertypes.AppOptions,
    evmAppOptions EVMOptionsFn, // Add this parameter
    baseAppOptions ...func(*baseapp.BaseApp),
) *ChainApp {
    // ...
}
```

#### Replace SDK Encoding <a href="#replace-sdk-encoding" id="replace-sdk-encoding"></a>

```
encodingConfig := evmencoding.MakeConfig()
appCodec := encodingConfig.Codec
legacyAmino := encodingConfig.Amino
txConfig := encodingConfig.TxConfig
```

#### Add Store Keys <a href="#add-store-keys" id="add-store-keys"></a>

```
keys := storetypes.NewKVStoreKeys(
    // ... existing keys
    evmtypes.StoreKey,
    feemarkettypes.StoreKey,
    erc20types.StoreKey,
)

tkeys := storetypes.NewTransientStoreKeys(
    paramstypes.TStoreKey,
    evmtypes.TransientKey,
    feemarkettypes.TransientKey,
)
```

#### Initialize Keepers (Critical Order) <a href="#initialize-keepers-critical-order" id="initialize-keepers-critical-order"></a>

Keepers must be initialized in exact order: FeeMarket → EVM → Erc20 → Transfer

```
// Initialize AuthzKeeper if not already done
app.AuthzKeeper = authzkeeper.NewKeeper(
    keys[authz.StoreKey],
    appCodec,
    app.MsgServiceRouter(),
    app.AccountKeeper,
)

// Initialize FeeMarketKeeper
app.FeeMarketKeeper = feemarketkeeper.NewKeeper(
    appCodec,
    authtypes.NewModuleAddress(govtypes.ModuleName),
    keys[feemarkettypes.StoreKey],
    tkeys[feemarkettypes.TransientKey],
    app.GetSubspace(feemarkettypes.ModuleName),
)

// Initialize EVMKeeper
tracer := cast.ToString(appOpts.Get(srvflags.EVMTracer))
app.EVMKeeper = evmkeeper.NewKeeper(
    appCodec,
    keys[evmtypes.StoreKey],
    tkeys[evmtypes.TransientKey],
    authtypes.NewModuleAddress(govtypes.ModuleName),
    app.AccountKeeper,
    app.BankKeeper,
    app.StakingKeeper,
    app.FeeMarketKeeper,
    &app.Erc20Keeper, // Pass pointer for circular dependency
    tracer,
    app.GetSubspace(evmtypes.ModuleName),
)

// Initialize Erc20Keeper
app.Erc20Keeper = erc20keeper.NewKeeper(
    keys[erc20types.StoreKey],
    appCodec,
    authtypes.NewModuleAddress(govtypes.ModuleName),
    app.AccountKeeper,
    app.BankKeeper,
    app.EVMKeeper,
    app.StakingKeeper,
    app.AuthzKeeper,
    &app.TransferKeeper, // Pass pointer for circular dependency
)

// Initialize extended TransferKeeper
app.TransferKeeper = ibctransferkeeper.NewKeeper(
    appCodec,
    keys[ibctransfertypes.StoreKey],
    app.GetSubspace(ibctransfertypes.ModuleName),
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.ChannelKeeper,
    app.IBCKeeper.PortKeeper,
    app.AccountKeeper,
    app.BankKeeper,
    scopedTransferKeeper,
    app.Erc20Keeper,
    authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)

// CRITICAL: Wire IBC callbacks for automatic ERC20 registration
transferModule := transfer.NewIBCModule(app.TransferKeeper)
app.Erc20Keeper.SetICS20Module(transferModule)

// Configure EVM Precompiles
corePrecompiles := NewAvailableStaticPrecompiles(
    *app.StakingKeeper,
    app.DistrKeeper,
    app.BankKeeper,
    app.Erc20Keeper,
    app.AuthzKeeper,
    app.TransferKeeper,
    app.IBCKeeper.ChannelKeeper,
    app.EVMKeeper,
    app.GovKeeper,
    app.SlashingKeeper,
    app.EvidenceKeeper,
)
app.EVMKeeper.WithStaticPrecompiles(corePrecompiles)
```

#### Add Modules to Module Manager <a href="#add-modules-to-module-manager" id="add-modules-to-module-manager"></a>

```
app.ModuleManager = module.NewManager(
    // ... existing modules
    evm.NewAppModule(app.EVMKeeper, app.AccountKeeper, app.GetSubspace(evmtypes.ModuleName)),
    feemarket.NewAppModule(app.FeeMarketKeeper, app.GetSubspace(feemarkettypes.ModuleName)),
    erc20.NewAppModule(app.Erc20Keeper, app.AccountKeeper, app.GetSubspace(erc20types.ModuleName)),
    transfer.NewAppModule(app.TransferKeeper),
)
```

#### Update Module Ordering <a href="#update-module-ordering" id="update-module-ordering"></a>

```
// SetOrderBeginBlockers - EVM must come after feemarket
app.ModuleManager.SetOrderBeginBlockers(
    // ... other modules
    erc20types.ModuleName,
    feemarkettypes.ModuleName,
    evmtypes.ModuleName,
    // ...
)

// SetOrderEndBlockers
app.ModuleManager.SetOrderEndBlockers(
    // ... other modules
    evmtypes.ModuleName,
    feemarkettypes.ModuleName,
    erc20types.ModuleName,
    // ...
)

// SetOrderInitGenesis - feemarket must be before genutil
genesisModuleOrder := []string{
    // ... other modules
    evmtypes.ModuleName,
    feemarkettypes.ModuleName,
    erc20types.ModuleName,
    // ...
}
```

#### Update Ante Handler <a href="#update-ante-handler" id="update-ante-handler"></a>

```
options := ante.HandlerOptions{
    AccountKeeper:          app.AccountKeeper,
    BankKeeper:             app.BankKeeper,
    SignModeHandler:        txConfig.SignModeHandler(),
    FeegrantKeeper:         app.FeeGrantKeeper,
    SigGasConsumer:         ante.DefaultSigVerificationGasConsumer,
    FeeMarketKeeper:        app.FeeMarketKeeper,
    EvmKeeper:              app.EVMKeeper,
    ExtensionOptionChecker: Ontomirevmtypes.HasDynamicFeeExtensionOption,
    MaxTxGasWanted:         cast.ToUint64(appOpts.Get(srvflags.EVMMaxTxGasWanted)),
    TxFeeChecker:           evmevmante.NewDynamicFeeChecker(app.FeeMarketKeeper),
    // ... other options
}

anteHandler, err := ante.NewAnteHandler(options)
if err != nil {
    panic(err)
}
app.SetAnteHandler(anteHandler)
```

#### Update DefaultGenesis <a href="#update-defaultgenesis" id="update-defaultgenesis"></a>

```
func (a *ChainApp) DefaultGenesis() map[string]json.RawMessage {
    genesis := a.BasicModuleManager.DefaultGenesis(a.appCodec)

    // Add EVM genesis config
    evmGenState := evmtypes.DefaultGenesisState()
    evmGenState.Params.ActiveStaticPrecompiles = evmtypes.AvailableStaticPrecompiles
    genesis[evmtypes.ModuleName] = a.appCodec.MustMarshalJSON(evmGenState)

    // Add ERC20 genesis config
    erc20GenState := erc20types.DefaultGenesisState()
    genesis[erc20types.ModuleName] = a.appCodec.MustMarshalJSON(erc20GenState)

    return genesis
}
```

### Step 8: Create Ante Handler Files <a href="#step-8-create-ante-handler-files" id="step-8-create-ante-handler-files"></a>

Create new `ante/` directory in your project root.

#### ante/handler\_options.go <a href="#antehandler_optionsgo" id="antehandler_optionsgo"></a>

```
package ante

import (
    errorsmod "Ontomirsdk.io/errors"
    storetypes "Ontomirsdk.io/store/types"
    txsigning "Ontomirsdk.io/x/tx/signing"

    "github.com/Ontomir/Ontomir-sdk/codec"
    errortypes "github.com/Ontomir/Ontomir-sdk/types/errors"
    "github.com/Ontomir/Ontomir-sdk/x/auth/ante"
    "github.com/Ontomir/Ontomir-sdk/x/auth/signing"
    authtypes "github.com/Ontomir/Ontomir-sdk/x/auth/types"

    anteinterfaces "github.com/Ontomir/evm/ante/interfaces"
    ibckeeper "github.com/Ontomir/ibc-go/v10/modules/core/keeper"
)

type HandlerOptions struct {
    Cdc                    codec.BinaryCodec
    AccountKeeper          anteinterfaces.AccountKeeper
    BankKeeper             anteinterfaces.BankKeeper
    IBCKeeper              *ibckeeper.Keeper
    FeeMarketKeeper        anteinterfaces.FeeMarketKeeper
    EvmKeeper              anteinterfaces.EVMKeeper
    FeegrantKeeper         ante.FeegrantKeeper
    ExtensionOptionChecker ante.ExtensionOptionChecker
    SignModeHandler        *txsigning.HandlerMap
    SigGasConsumer         func(meter storetypes.GasMeter, sig signing.SignatureV2, params authtypes.Params) error
    MaxTxGasWanted         uint64
    TxFeeChecker           ante.TxFeeChecker
}

func (options HandlerOptions) Validate() error {
    if options.Cdc == nil {
        return errorsmod.Wrap(errortypes.ErrLogic, "codec is required for ante builder")
    }
    if options.AccountKeeper == nil {
        return errorsmod.Wrap(errortypes.ErrLogic, "account keeper is required for ante builder")
    }
    if options.BankKeeper == nil {
        return errorsmod.Wrap(errortypes.ErrLogic, "bank keeper is required for ante builder")
    }
    if options.SignModeHandler == nil {
        return errorsmod.Wrap(errortypes.ErrLogic, "sign mode handler is required for ante builder")
    }
    if options.EvmKeeper == nil {
        return errorsmod.Wrap(errortypes.ErrLogic, "evm keeper is required for ante builder")
    }
    if options.FeeMarketKeeper == nil {
        return errorsmod.Wrap(errortypes.ErrLogic, "feemarket keeper is required for ante builder")
    }
    return nil
}
```

#### ante/ante\_Ontomir.go <a href="#anteante_ontomirgo" id="anteante_ontomirgo"></a>

```
package ante

import (
    sdk "github.com/Ontomir/Ontomir-sdk/types"
    "github.com/Ontomir/Ontomir-sdk/x/auth/ante"
    ibcante "github.com/Ontomir/ibc-go/v10/modules/core/ante"

    Ontomirante "github.com/Ontomir/evm/ante/Ontomir"
)

// newOntomirAnteHandler creates the default SDK ante handler for Ontomir transactions
func newOntomirAnteHandler(options HandlerOptions) sdk.AnteHandler {
    return sdk.ChainAnteDecorators(
        ante.NewSetUpContextDecorator(),
        ante.NewExtensionOptionsDecorator(options.ExtensionOptionChecker),
        Ontomirante.NewValidateBasicDecorator(options.EvmKeeper),
        ante.NewTxTimeoutHeightDecorator(),
        ante.NewValidateMemoDecorator(options.AccountKeeper),
        ante.NewConsumeGasForTxSizeDecorator(options.AccountKeeper),
        Ontomirante.NewDeductFeeDecorator(
            options.AccountKeeper,
            options.BankKeeper,
            options.FeegrantKeeper,
            options.TxFeeChecker,
        ),
        ante.NewSetPubKeyDecorator(options.AccountKeeper),
        ante.NewValidateSigCountDecorator(options.AccountKeeper),
        ante.NewSigGasConsumeDecorator(options.AccountKeeper, options.SigGasConsumer),
        ante.NewSigVerificationDecorator(options.AccountKeeper, options.SignModeHandler),
        ante.NewIncrementSequenceDecorator(options.AccountKeeper),
        ibcante.NewRedundantRelayDecorator(options.IBCKeeper),
        Ontomirante.NewGasWantedDecorator(options.EvmKeeper, options.FeeMarketKeeper),
    )
}
```

#### ante/ante\_evm.go <a href="#anteante_evmgo" id="anteante_evmgo"></a>

```
package ante

import (
    sdk "github.com/Ontomir/Ontomir-sdk/types"
    evmante "github.com/Ontomir/evm/ante/evm"
)

// newMonoEVMAnteHandler creates the sdk.AnteHandler for EVM transactions
func newMonoEVMAnteHandler(options HandlerOptions) sdk.AnteHandler {
    return sdk.ChainAnteDecorators(
        evmante.NewEVMMonoDecorator(
            options.AccountKeeper,
            options.FeeMarketKeeper,
            options.EvmKeeper,
            options.MaxTxGasWanted,
        ),
    )
}
```

#### ante/ante.go <a href="#anteantego" id="anteantego"></a>

```
package ante

import (
    errorsmod "Ontomirsdk.io/errors"
    sdk "github.com/Ontomir/Ontomir-sdk/types"
    errortypes "github.com/Ontomir/Ontomir-sdk/types/errors"
    authante "github.com/Ontomir/Ontomir-sdk/x/auth/ante"
    "github.com/Ontomir/evm/ante/evm"
)

// NewAnteHandler routes Ethereum or SDK transactions to the appropriate handler
func NewAnteHandler(options HandlerOptions) (sdk.AnteHandler, error) {
    if err := options.Validate(); err != nil {
        return nil, err
    }

    return func(ctx sdk.Context, tx sdk.Tx, sim bool) (newCtx sdk.Context, err error) {
        var anteHandler sdk.AnteHandler

        if ethTx, ok := tx.(*evm.EthTx); ok {
            // Handle as Ethereum transaction
            anteHandler = newMonoEVMAnteHandler(options)
        } else {
            // Handle as normal Ontomir SDK transaction
            anteHandler = newOntomirAnteHandler(options)
        }

        return anteHandler(ctx, tx, sim)
    }, nil
}
```

### Step 9: Update Command Files <a href="#step-9-update-command-files" id="step-9-update-command-files"></a>

#### Update cmd/evmd/commands.go <a href="#update-cmdevmdcommandsgo" id="update-cmdevmdcommandsgo"></a>

```
import (
    // Add imports
    evmcmd "github.com/Ontomir/evm/client"
    evmserver "github.com/Ontomir/evm/server"
    evmserverconfig "github.com/Ontomir/evm/server/config"
    srvflags "github.com/Ontomir/evm/server/flags"
)

// Define custom app config struct
type CustomAppConfig struct {
    serverconfig.Config
    EVM     evmserverconfig.EVMConfig
    JSONRPC evmserverconfig.JSONRPCConfig
    TLS     evmserverconfig.TLSConfig
}

// Update initAppConfig to include EVM config
func initAppConfig() (string, interface{}) {
    srvCfg, customAppTemplate := serverconfig.AppConfig(DefaultDenom)
    customAppConfig := CustomAppConfig{
        Config:  *srvCfg,
        EVM:     *evmserverconfig.DefaultEVMConfig(),
        JSONRPC: *evmserverconfig.DefaultJSONRPCConfig(),
        TLS:     *evmserverconfig.DefaultTLSConfig(),
    }
    customAppTemplate += evmserverconfig.DefaultEVMConfigTemplate
    return customAppTemplate, customAppConfig
}

// In initRootCmd, replace server.AddCommands with evmserver.AddCommands
func initRootCmd(...) {
    // ...
    evmserver.AddCommands(
        rootCmd,
        evmserver.NewDefaultStartOptions(newApp, app.DefaultNodeHome),
        appExport,
        addModuleInitFlags,
    )

    rootCmd.AddCommand(
        // ... existing commands
        evmcmd.KeyCommands(app.DefaultNodeHome, true),
    )

    var err error
    rootCmd, err = srvflags.AddTxFlags(rootCmd)
    if err != nil {
        panic(err)
    }
}
```

#### Update cmd/evmd/root.go <a href="#update-cmdevmdrootgo" id="update-cmdevmdrootgo"></a>

```
import (
    // ... existing imports
    evmkeyring "github.com/Ontomir/evm/crypto/keyring"
    evmtypes "github.com/Ontomir/evm/x/vm/types"
    sdk "github.com/Ontomir/Ontomir-sdk/types"
    "github.com/Ontomir/Ontomir-sdk/client/flags"
)

func NewRootCmd() *cobra.Command {
    // ...
    // In client context setup:
    clientCtx = clientCtx.
        WithKeyringOptions(evmkeyring.Option()).
        WithBroadcastMode(flags.FlagBroadcastMode).
        WithLedgerHasProtobuf(true)

    // Update the coin type
    cfg := sdk.GetConfig()
    cfg.SetCoinType(evmtypes.Bip44CoinType) // Sets coin type to 60
    cfg.Seal()

    // ...
    return rootCmd
}
```

### Step 10: Sign Mode Configuration (Optional) <a href="#step-10-sign-mode-configuration-optional" id="step-10-sign-mode-configuration-optional"></a>

Sign Mode Textual is a new Ontomir SDK signing method that may not be compatible with all Ethereum signing workflows.

#### Option A: Disable Sign Mode Textual (Recommended for pure EVM compatibility) <a href="#option-a-disable-sign-mode-textual-recommended-for-pure-evm-compatibility" id="option-a-disable-sign-mode-textual-recommended-for-pure-evm-compatibility"></a>

```
// In app.go
import (
    authtx "github.com/Ontomir/Ontomir-sdk/x/auth/tx"
)

// ... in NewChainApp, where you set up your txConfig:
txConfig := authtx.NewTxConfigWithOptions(
    appCodec,
    authtx.ConfigOptions{
        // Remove SignMode_SIGN_MODE_TEXTUAL from enabled sign modes
        EnabledSignModes: []signing.SignMode{
            signing.SignMode_SIGN_MODE_DIRECT,
            signing.SignMode_SIGN_MODE_LEGACY_AMINO_JSON,
            signing.SignMode_SIGN_MODE_EIP_191,
        },
        // ...
    },
)
```

#### Option B: Enable Sign Mode Textual <a href="#option-b-enable-sign-mode-textual" id="option-b-enable-sign-mode-textual"></a>

If your chain requires Sign Mode Textual support, ensure your ante handler and configuration support it. The reference implementation in `evmd` enables it by default.

### Step 11: Testing Your Integration <a href="#step-11-testing-your-integration" id="step-11-testing-your-integration"></a>

#### Build and Run Tests <a href="#build-and-run-tests" id="build-and-run-tests"></a>

```
# Run all unit tests
make test-all

# Run EVM-specific tests
make test-evmd

# Run integration tests
make test-integration
```

#### Local Node Testing <a href="#local-node-testing" id="local-node-testing"></a>

```
# Copy and adapt the script from Ontomir EVM repo
curl -O https://raw.githubusercontent.com/Ontomir/evm/main/local_node.sh
chmod +x local_node.sh
./local_node.sh
```

#### Verify EVM Functionality <a href="#verify-evm-functionality" id="verify-evm-functionality"></a>

* Check JSON-RPC server starts on configured port (default: 8545)
* Verify MetaMask connection to your local node
* Test precompiles accessibility at expected addresses
* Confirm IBC tokens automatically register as ERC20s

#### Genesis Validation <a href="#genesis-validation" id="genesis-validation"></a>

```
evmd genesis validate-genesis
```

***

Check [github.com/Ontomir/evm](https://github.com/Ontomir/evm) for the latest updates or to open an issue.
