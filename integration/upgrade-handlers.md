# Upgrade Handlers

> Understanding and implementing on-chain upgrade handlers for coordinated chain upgrades

### Overview <a href="#overview" id="overview"></a>

Upgrade handlers enable coordinated on-chain upgrades across all validators at specific block heights via governance proposals. They provide a mechanism for chains to perform data migrations, parameter updates, and module upgrades in a deterministic way.

Upgrade handlers are critical for maintaining consensus during chain upgrades. All validators must run the same upgrade logic at the same height.

### When to Use Upgrade Handlers <a href="#when-to-use-upgrade-handlers" id="when-to-use-upgrade-handlers"></a>

Upgrade handlers are required when:

* **Breaking state changes**: Modifying storage formats or data structures
* **Module migrations**: Updating module versions or parameters
* **Protocol upgrades**: Implementing new features that require state transitions
* **Data migrations**: Moving data between different storage locations

### Basic Structure <a href="#basic-structure" id="basic-structure"></a>

#### Registering Upgrade Handlers <a href="#registering-upgrade-handlers" id="registering-upgrade-handlers"></a>

Upgrade handlers are registered in your app's `RegisterUpgradeHandlers()` method:

\`\`\`go // app/upgrades.go package app

import ( "context" upgradetypes "Ontomirsdk.io/x/upgrade/types" sdk "github.com/Ontomir/Ontomir-sdk/types" "github.com/Ontomir/Ontomir-sdk/types/module"

)

func (app \*App) RegisterUpgradeHandlers() { app.UpgradeKeeper.SetUpgradeHandler( "v1.0.0", // upgrade name (must match governance proposal) func(ctx context.Context, plan upgradetypes.Plan, fromVM module.VersionMap) (module.VersionMap, error) { sdkCtx := sdk.UnwrapSDKContext(ctx) sdkCtx.Logger().Info("Starting upgrade", "name", plan.Name) // Run module migrations migrations, err := app.ModuleManager.RunMigrations(ctx, app.configurator, fromVM) if err != nil { return nil, err } // Add custom migration logic here sdkCtx.Logger().Info("Upgrade complete", "name", plan.Name) return migrations, nil }, )

}

````
</Expandable>

### Module Version Management

The upgrade handler receives and returns a `module.VersionMap` that tracks module versions:

```go
// fromVM contains the module versions before the upgrade
// The returned VersionMap contains the new versions after migration
migrations, err := app.ModuleManager.RunMigrations(ctx, app.configurator, fromVM)
````

### Organizing Upgrade Code <a href="#organizing-upgrade-code" id="organizing-upgrade-code"></a>

For better maintainability, organize upgrades in separate packages:

#### Directory Structure <a href="#directory-structure" id="directory-structure"></a>

```
app/
├── upgrades/
│   ├── v1_0_0/
│   │   ├── constants.go    # Upgrade name and configuration
│   │   ├── handler.go      # Main upgrade handler
│   │   └── migrations.go   # Migration logic
│   ├── v1_1_0/
│   │   ├── constants.go
│   │   ├── handler.go
│   │   └── migrations.go
│   └── types.go            # Shared types
└── upgrades.go             # RegisterUpgradeHandlers
```

#### constants.go <a href="#constantsgo" id="constantsgo"></a>

\`\`\`go // app/upgrades/v1\_0\_0/constants.go package v1\_0\_0

import ( storetypes "Ontomirsdk.io/store/types" "github.com/yourchain/app/upgrades" )

const UpgradeName = "v1.0.0"

var Upgrade = upgrades.Upgrade{ UpgradeName: UpgradeName, CreateUpgradeHandler: CreateUpgradeHandler, StoreUpgrades: storetypes.StoreUpgrades{ Added: \[]string{}, // New modules Deleted: \[]string{}, // Removed modules }, }

````
</Expandable>

### handler.go

<Expandable title="View upgrade handler implementation">
```go
// app/upgrades/v1_0_0/handler.go
package v1_0_0

import (
    "context"

    storetypes "Ontomirsdk.io/store/types"
    upgradetypes "Ontomirsdk.io/x/upgrade/types"
    sdk "github.com/Ontomir/Ontomir-sdk/types"
    "github.com/Ontomir/Ontomir-sdk/types/module"
)

func CreateUpgradeHandler(
    mm *module.Manager,
    configurator module.Configurator,
    keepers *upgrades.UpgradeKeepers,
    storeKeys map[string]*storetypes.KVStoreKey,
) upgradetypes.UpgradeHandler {
    return func(c context.Context, plan upgradetypes.Plan, vm module.VersionMap) (module.VersionMap, error) {
        ctx := sdk.UnwrapSDKContext(c)

        // Run module migrations
        vm, err := mm.RunMigrations(c, configurator, vm)
        if err != nil {
            return nil, err
        }

        // Custom migrations
        if err := runCustomMigrations(ctx, keepers, storeKeys); err != nil {
            return nil, err
        }

        return vm, nil
    }
}
````

### Migration Patterns <a href="#migration-patterns" id="migration-patterns"></a>

#### Parameter Migrations <a href="#parameter-migrations" id="parameter-migrations"></a>

Migrating module parameters to new formats:

```
func migrateParams(ctx sdk.Context, keeper paramskeeper.Keeper) error {
    // Get old params
    var oldParams v1.Params
    keeper.GetParamSet(ctx, &oldParams)

    // Convert to new format
    newParams := v2.Params{
        Field1: oldParams.Field1,
        Field2: convertField(oldParams.Field2),
        // New field with default value
        Field3: "default",
    }

    // Set new params
    keeper.SetParams(ctx, newParams)
    return nil
}
```

#### State Migrations <a href="#state-migrations" id="state-migrations"></a>

Moving data between different storage locations:

```
func migrateState(ctx sdk.Context, storeKey storetypes.StoreKey) error {
    store := ctx.KVStore(storeKey)

    // Iterate over old storage
    iterator := storetypes.KVStorePrefixIterator(store, oldPrefix)
    defer iterator.Close()

    for ; iterator.Valid(); iterator.Next() {
        oldKey := iterator.Key()
        value := iterator.Value()

        // Transform key/value if needed
        newKey := transformKey(oldKey)
        newValue := transformValue(value)

        // Write to new location
        store.Set(newKey, newValue)

        // Delete old entry
        store.Delete(oldKey)
    }

    return nil
}
```

#### Module Addition/Removal <a href="#module-additionremoval" id="module-additionremoval"></a>

Adding or removing modules during upgrade:

```
// In constants.go
var Upgrade = upgrades.Upgrade{
    UpgradeName: "v2.0.0",
    CreateUpgradeHandler: CreateUpgradeHandler,
    StoreUpgrades: storetypes.StoreUpgrades{
        Added:   []string{"newmodule"},
        Deleted: []string{"oldmodule"},
    },
}

// In handler.go
func CreateUpgradeHandler(...) upgradetypes.UpgradeHandler {
    return func(c context.Context, plan upgradetypes.Plan, vm module.VersionMap) (module.VersionMap, error) {
        // Delete old module version
        delete(vm, "oldmodule")

        // Initialize new module
        if err := newModuleKeeper.InitGenesis(ctx, defaultGenesis); err != nil {
            return nil, err
        }

        // Run migrations
        return mm.RunMigrations(c, configurator, vm)
    }
}
```

### Best Practices <a href="#best-practices" id="best-practices"></a>

Always test upgrade handlers thoroughly on testnets before mainnet deployment.

#### Idempotency <a href="#idempotency" id="idempotency"></a>

Make migrations idempotent when possible:

```
func migrateSomething(ctx sdk.Context, store sdk.KVStore) error {
    // Check if migration already done
    if store.Has(migrationCompleteKey) {
        ctx.Logger().Info("Migration already completed, skipping")
        return nil
    }

    // Perform migration
    // ...

    // Mark as complete
    store.Set(migrationCompleteKey, []byte{1})
    return nil
}
```

#### Error Handling <a href="#error-handling" id="error-handling"></a>

Use comprehensive error handling and logging:

```
func migrate(ctx sdk.Context, keeper Keeper) error {
    ctx.Logger().Info("Starting migration", "module", "mymodule")

    count := 0
    iterator := keeper.IterateAllRecords(ctx)
    defer iterator.Close()

    for ; iterator.Valid(); iterator.Next() {
        if err := processRecord(iterator.Key(), iterator.Value()); err != nil {
            ctx.Logger().Error("Failed to migrate record",
                "key", iterator.Key(),
                "error", err,
            )
            return fmt.Errorf("migration failed at record %d: %w", count, err)
        }
        count++

        // Log progress for long migrations
        if count%1000 == 0 {
            ctx.Logger().Info("Migration progress", "processed", count)
        }
    }

    ctx.Logger().Info("Migration complete", "total_migrated", count)
    return nil
}
```

#### Testing <a href="#testing" id="testing"></a>

Create comprehensive tests for upgrade handlers:

```
func TestUpgradeHandler(t *testing.T) {
    app := setupApp(t)
    ctx := app.NewContext(false, tmproto.Header{Height: 1})

    // Setup pre-upgrade state
    setupOldState(t, ctx, app)

    // Run upgrade handler
    _, err := v1_0_0.CreateUpgradeHandler(
        app.ModuleManager,
        app.configurator,
        &upgrades.UpgradeKeepers{
            // ... keepers
        },
        app.keys,
    )(ctx, upgradetypes.Plan{Name: "v1.0.0"}, app.ModuleManager.GetVersionMap())

    require.NoError(t, err)

    // Verify post-upgrade state
    verifyNewState(t, ctx, app)
}
```

### Upgrade Process <a href="#upgrade-process" id="upgrade-process"></a>

#### Create Upgrade Proposal <a href="#create-upgrade-proposal" id="create-upgrade-proposal"></a>

Submit a governance proposal with the upgrade details:

```
mantrachaind tx gov submit-proposal software-upgrade v1.0.0 \
    --title "Upgrade to v1.0.0" \
    --description "Upgrade description" \
    --upgrade-height 1000000 \
    --from validator \
    --deposit 10000000stake
```

#### Vote on Proposal <a href="#vote-on-proposal" id="vote-on-proposal"></a>

Validators and delegators vote on the upgrade:

```
mantrachaind tx gov vote 1 yes --from validator
```

#### Prepare Binary <a href="#prepare-binary" id="prepare-binary"></a>

Build and distribute the new binary with the upgrade handler:

```
# Build new binary
make build

# Test upgrade on local network
./scripts/test-upgrade.sh

# Distribute to validators
# Use Cosmovisor for automated upgrades
```

#### Monitor Upgrade <a href="#monitor-upgrade" id="monitor-upgrade"></a>

Watch logs during the upgrade height:

```
# Monitor upgrade logs
tail -f ~/.mantrachaind/logs/upgrade.log

# Verify upgrade success
mantrachaind query upgrade applied v1.0.0
```

### Ontomir EVM Specific Migrations <a href="#ontomir-evm-specific-migrations" id="ontomir-evm-specific-migrations"></a>

For Ontomir EVM chains, specific migrations include:

* **ERC20 Precompiles Migration**: Required for v0.3.x to v0.4.0
* **Fee Market Parameters**: Updating EIP-1559 parameters
* **Custom Precompiles**: Registering new precompiled contracts
* **EVM State**: Migrating account balances or contract storage

### Troubleshooting <a href="#troubleshooting" id="troubleshooting"></a>

#### Consensus Failure <a href="#consensus-failure" id="consensus-failure"></a>

**Symptom:** Chain halts with consensus failure at upgrade height

**Causes:**

* Validators running different binary versions
* Upgrade handler not registered
* Non-deterministic migration logic

**Solution:**

* Ensure all validators have the same binary
* Verify upgrade handler is registered
* Review migration logic for non-determinism

#### Upgrade Panic <a href="#upgrade-panic" id="upgrade-panic"></a>

**Symptom:** Node panics during upgrade

**Causes:**

* Unhandled error in migration
* Missing required state
* Invalid type assertions

**Solution:**

* Add comprehensive error handling
* Validate state before migration
* Use safe type conversions

#### State Corruption <a href="#state-corruption" id="state-corruption"></a>

**Symptom:** Invalid state after upgrade

**Causes:**

* Partial migration completion
* Incorrect data transformation
* Missing cleanup of old data

**Solution:**

* Make migrations atomic
* Thoroughly test transformations
* Ensure old data is properly cleaned up

### References <a href="#references" id="references"></a>

* [Ontomir SDK Upgrade Module](https://docs.ontomir.network/main/build/modules/upgrade)
* [Cosmovisor Documentation](https://docs.ontomir.network/main/build/tooling/cosmovisor)
