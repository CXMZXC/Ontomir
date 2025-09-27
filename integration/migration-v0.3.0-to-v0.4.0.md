# Migration v0.3.0 to v0.4.0

> Upgrading Ontomir EVM from v0.3.x to v0.4.0, step by step

### Process Overview <a href="#process-overview" id="process-overview"></a>

* Create an upgrade branch and freeze schema-affecting changes
* Export a pre-upgrade state and archive node configs
* Bump `Ontomir/evm` to v0.4.0 and align Ontomir SDK/IBC/CometBFT constraints
* Rewire keepers and AppModule (imports, constructors, `RegisterServices`)
* Add client context field and `SetClientCtx` method
* Add pending transaction listener support
* **Migrate ERC20 precompiles if you have existing token pairs** (see \[this section]./(erc20-precompiles-migration))
* Audit and migrate EVM & FeeMarket params (EIP-1559 knobs, denom/decimals)
* Implement store/params migrations in your UpgradeHandler

### Prep <a href="#prep" id="prep"></a>

* Create a branch: `git switch -c upgrade/evm-v0.4`
* Ensure a clean build + tests green pre-upgrade
* Snapshot your current params/genesis for comparison later

```
git switch -c upgrade/evm-v0.4
go test ./...
evmd export > pre-upgrade-genesis.json
```

### Dependency bumps <a href="#dependency-bumps" id="dependency-bumps"></a>

#### Pin EVM and tidy <a href="#pin-evm-and-tidy" id="pin-evm-and-tidy"></a>

Bump the `Ontomir/evm` dependency in `go.mod`:

```
- github.com/Ontomir/evm v0.3.1
+ github.com/Ontomir/evm v0.4.0
```

#### Transitive bumps <a href="#transitive-bumps" id="transitive-bumps"></a>

Check for minor dependency bumps (e.g., `google.golang.org/protobuf`, `github.com/gofrs/flock`, `github.com/consensys/gnark-crypto`):

```
go mod tidy
```

Resolve any version conflicts here before moving on.

### App constructor return type & CLI command wiring <a href="#app-constructor-return-type-andamp-cli-command-wiring" id="app-constructor-return-type-andamp-cli-command-wiring"></a>

Update your app's `newApp` to return an `evmserver.Application` rather than `servertypes.Application`, and CLI commands that still expect an SDK app creator require a wrapper.

#### Change the return type <a href="#change-the-return-type" id="change-the-return-type"></a>

```
// cmd/myapp/cmd/root.go
import (
    evmserver "github.com/Ontomir/evm/server"
)

func (a appCreator) newApp(
    l log.Logger,
    db dbm.DB,
    traceStore io.Writer,
    appOpts servertypes.AppOptions,
) evmserver.Application { // Changed from servertypes.Application
    // ...
}
```

#### Provide a wrapper for commands that expect the SDK type <a href="#provide-a-wrapper-for-commands-that-expect-the-sdk-type" id="provide-a-wrapper-for-commands-that-expect-the-sdk-type"></a>

Create a thin wrapper and use it for `pruning.Cmd` and `snapshot.Cmd`:

```
// cmd/myapp/cmd/root.go
sdkAppCreatorWrapper := func(l log.Logger, d dbm.DB, w io.Writer, ao servertypes.AppOptions) servertypes.Application {
    return ac.newApp(l, d, w, ao)
}

rootCmd.AddCommand(
    pruning.Cmd(sdkAppCreatorWrapper, myapp.DefaultNodeHome),
    snapshot.Cmd(sdkAppCreatorWrapper),
)
```

#### Add clientCtx and SetClientCtx <a href="#add-clientctx-and-setclientctx" id="add-clientctx-and-setclientctx"></a>

Add the clientCtx to your app object:

```
// app/app.go
import (
    "github.com/Ontomir/Ontomir-sdk/client"
)

type MyApp struct {
    // ... existing fields
    clientCtx client.Context
}

func (app *MyApp) SetClientCtx(clientCtx client.Context) {
    app.clientCtx = clientCtx
}
```

### Pending-tx listener support <a href="#pending-tx-listener-support" id="pending-tx-listener-support"></a>

#### Imports <a href="#imports" id="imports"></a>

Import the EVM ante package and geth common:

```
// app/app.go
import (
    "github.com/Ontomir/evm/ante"
    "github.com/ethereum/go-ethereum/common"
)
```

#### App state: listeners slice <a href="#app-state-listeners-slice" id="app-state-listeners-slice"></a>

Add a new field for listeners:

```
// app/app.go
type MyApp struct {
    // ... existing fields
    pendingTxListeners []ante.PendingTxListener
}
```

#### Registration method <a href="#registration-method" id="registration-method"></a>

Add a public method to register a listener by txHash:

```
// app/app.go
func (app *MyApp) RegisterPendingTxListener(listener func(common.Hash)) {
    app.pendingTxListeners = append(app.pendingTxListeners, listener)
}
```

### Precompiles: optionals + codec injection <a href="#precompiles-optionals--codec-injection" id="precompiles-optionals--codec-injection"></a>

#### New imports <a href="#new-imports" id="new-imports"></a>

```
// app/keepers/precompiles.go
import (
    "Ontomirsdk.io/core/address"
    addresscodec "github.com/Ontomir/Ontomir-sdk/codec/address"
    sdk "github.com/Ontomir/Ontomir-sdk/types"
)
```

#### Define Optionals + defaults + functional options <a href="#define-optionals--defaults--functional-options" id="define-optionals--defaults--functional-options"></a>

Create a small options container with sane defaults pulled from the app's bech32 config:

```
// app/keepers/precompiles.go
type Optionals struct {
    AddressCodec       address.Codec // used by gov/staking
    ValidatorAddrCodec address.Codec // used by slashing
    ConsensusAddrCodec address.Codec // used by slashing
}

func defaultOptionals() Optionals {
    return Optionals{
        AddressCodec:       addresscodec.NewBech32Codec(sdk.GetConfig().GetBech32AccountAddrPrefix()),
        ValidatorAddrCodec: addresscodec.NewBech32Codec(sdk.GetConfig().GetBech32ValidatorAddrPrefix()),
        ConsensusAddrCodec: addresscodec.NewBech32Codec(sdk.GetConfig().GetBech32ConsensusAddrPrefix()),
    }
}

type Option func(*Optionals)

func WithAddressCodec(c address.Codec) Option {
    return func(o *Optionals) { o.AddressCodec = c }
}

func WithValidatorAddrCodec(c address.Codec) Option {
    return func(o *Optionals) { o.ValidatorAddrCodec = c }
}

func WithConsensusAddrCodec(c address.Codec) Option {
    return func(o *Optionals) { o.ConsensusAddrCodec = c }
}
```

#### 4.3 Update the precompile factory to accept options <a href="#id-43-update-the-precompile-factory-to-accept-options" id="id-43-update-the-precompile-factory-to-accept-options"></a>

```
// app/keepers/precompiles.go
func NewAvailableStaticPrecompiles(
    ctx context.Context,
    // ... other params
    opts ...Option,
) map[common.Address]vm.PrecompiledContract {
    options := defaultOptionals()
    for _, opt := range opts {
        opt(&options)
    }
    // ... rest of implementation
}
```

#### 4.4 Modify individual precompile constructors <a href="#id-44-modify-individual-precompile-constructors" id="id-44-modify-individual-precompile-constructors"></a>

**ICS-20 precompile** now needs `bankKeeper` first:

```
- ibcTransferPrecompile, err := ics20precompile.NewPrecompile(
-     stakingKeeper,
+ ibcTransferPrecompile, err := ics20precompile.NewPrecompile(
+     bankKeeper,
+     stakingKeeper,
      transferKeeper,
      &channelKeeper,
      // ...
```

**Gov precompile** now requires an `AddressCodec`:

```
- govPrecompile, err := govprecompile.NewPrecompile(govKeeper, cdc)
+ govPrecompile, err := govprecompile.NewPrecompile(govKeeper, cdc, options.AddressCodec)
```

### ERC20 Precompiles Migration <a href="#erc20-precompiles-migration" id="erc20-precompiles-migration"></a>

\*\*This migration is required for chains with existing ERC20 token pairs\*\*

The storage mechanism for ERC20 precompiles has fundamentally changed in v0.4.0. Without proper migration, your ERC20 tokens will become inaccessible via EVM.

Include this migration with your upgrade if your chain has:

* IBC tokens converted to ERC20
* Token factory tokens with ERC20 representations
* Any existing `DynamicPrecompiles` or `NativePrecompiles` in storage

#### Implementation

For complete migration instructions, see: **ERC20 Precompiles Migration Guide**

Add this to your upgrade handler:

```
// In your upgrade handler
store := ctx.KVStore(storeKeys[erc20types.StoreKey])
const addressLength = 42

// Migrate dynamic precompiles
if oldData := store.Get([]byte("DynamicPrecompiles")); len(oldData) > 0 {
    for i := 0; i < len(oldData); i += addressLength {
        address := common.HexToAddress(string(oldData[i : i+addressLength]))
        erc20Keeper.SetDynamicPrecompile(ctx, address)
    }
    store.Delete([]byte("DynamicPrecompiles"))
}

// Migrate native precompiles
if oldData := store.Get([]byte("NativePrecompiles")); len(oldData) > 0 {
    for i := 0; i < len(oldData); i += addressLength {
        address := common.HexToAddress(string(oldData[i : i+addressLength]))
        erc20Keeper.SetNativePrecompile(ctx, address)
    }
    store.Delete([]byte("NativePrecompiles"))
}
```

#### Verification <a href="#verification" id="verification"></a>

Post-upgrade, verify your migration succeeded:

```
# Check ERC20 balance (should NOT be 0 if tokens existed before)
cast call $TOKEN_ADDRESS "balanceOf(address)" $USER_ADDRESS --rpc-url http://localhost:8545

# Verify precompiles in state
mantrachaind export | jq '.app_state.erc20.dynamic_precompiles'
```

### Build & quick tests <a href="#build-andamp-quick-tests" id="build-andamp-quick-tests"></a>

1.  **Compile**:

    ```
    go build ./...
    ```
2. **Smoke tests** (local single-node):
   * Start your node; ensure RPC starts cleanly
   * Deploy a trivial contract; verify events and logs
   * Send a couple 1559 txs and confirm base-fee behavior looks sane
   * (Optional) register a pending-tx listener and log hashes as they enter the mempool

### Rollout checklist <a href="#rollout-checklist" id="rollout-checklist"></a>

* Package the new binary (and Cosmovisor upgrade folder if you use it)
* Confirm all validators build the same commit (no `replace` lines)
* Share an `app.toml` diff only if you changed defaults; otherwise regenerate the file from the new binary and re-apply customizations
* Post-upgrade: monitor mempool/pending tx logs, base-fee progression, and contract events for the first 20-50 blocks

### Pitfalls & remedies <a href="#pitfalls-andamp-remedies" id="pitfalls-andamp-remedies"></a>

* **Forgot wrapper for CLI commands** → `pruning`/`snapshot` panic or wrong type:
  * Ensure you pass `sdkAppCreatorWrapper` (not `ac.newApp`) into those commands
* **ICS-20 precompile build error**:
  * You likely didn't pass `bankKeeper` first; update the call site
* **Governance precompile address parsing fails**:
  * Provide the correct `AddressCodec` via defaults or `WithAddressCodec(...)`
* **Listeners never fire**:
  * Register with `RegisterPendingTxListener` during app construction or module init

### Minimal code snippets <a href="#minimal-code-snippets" id="minimal-code-snippets"></a>

**App listeners**

```
// app/app.go
import (
    "github.com/Ontomir/evm/ante"
    "github.com/ethereum/go-ethereum/common"
)

type MyApp struct {
    // ...
    pendingTxListeners []ante.PendingTxListener
}

func (app *MyApp) RegisterPendingTxListener(l func(common.Hash)) {
    app.pendingTxListeners = append(app.pendingTxListeners, l)
}
```

**CLI wrapper**

```
// cmd/myapp/cmd/root.go
sdkAppCreatorWrapper := func(l log.Logger, d dbm.DB, w io.Writer, ao servertypes.AppOptions) servertypes.Application {
    return ac.newApp(l, d, w, ao)
}

rootCmd.AddCommand(
    pruning.Cmd(sdkAppCreatorWrapper, myapp.DefaultNodeHome),
    snapshot.Cmd(sdkAppCreatorWrapper),
)
```

**Precompile options & usage**

```
// app/keepers/precompiles.go
opts := []Option{
    // override defaults only if you use non-standard prefixes/codecs
    WithAddressCodec(myAcctCodec),
    WithValidatorAddrCodec(myValCodec),
    WithConsensusAddrCodec(myConsCodec),
}

pcs := NewAvailableStaticPrecompiles(ctx, /* ... keepers ... */, opts...)
```

### Verify before tagging <a href="#verify-before-tagging" id="verify-before-tagging"></a>

* `go.mod` has no `replace` lines for `github.com/Ontomir/evm`
* Node boots with expected RPC namespaces
* Contracts deploy/call; events stream; fee market behaves
* (If applicable) ICS-20 transfers work and precompiles execute
