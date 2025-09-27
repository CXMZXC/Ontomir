# PreciseBank

> Extended precision wrapper for x/bank enabling 18 decimal support

The `x/precisebank` module from Ontomir evm extends the standard `x/bank` module from 6 to 18 decimal precision for EVM compatibility.

For conceptual understanding of precision handling and mathematical proofs, see \[Precision Handling]\(/docs/evm/next/documentation/concepts/precision-handling).

### Overview <a href="#overview" id="overview"></a>

The module acts as a wrapper around `x/bank`, providing:

* **18 decimal precision** for EVM (10^18 sub-atomic units)
* **Backward compatibility** with 6 decimal Ontomir operations
* **Transparent conversion** between precision levels
* **Fractional balance tracking** for sub-test amounts

Developed with contributions from the \[Kava]\(https://www.kava.io/) team.

### State <a href="#state" id="state"></a>

The module maintains fractional balances and remainder :

| Object              | Key              | Value      | Description                               |
| ------------------- | ---------------- | ---------- | ----------------------------------------- |
| `FractionalBalance` | `0x01 + address` | `math.Int` | Account fractional balance (0 to 10^12-1) |
| `Remainder`         | `0x02`           | `math.Int` | Uncirculated fractional amount            |

#### Balance Representation <a href="#balance-representation" id="balance-representation"></a>

Full balance calculation:

```
atest_balance = test_balance × 10^12 + fractional_balance
```

Where:

* `test_balance`: Stored in x/bank (6 decimals)
* `fractional_balance`: Stored in x/precisebank (0 to 10^12-1)
* `atest_balance`: Full 18-decimal precision

### Keeper Interface <a href="#keeper-interface" id="keeper-interface"></a>

The module provides a bank-compatible keeper :

```
type Keeper interface {
    // Query methods
    GetBalance(ctx, addr, denom) sdk.Coin
    SpendableCoins(ctx, addr) sdk.Coins

    // Transfer methods
    SendCoins(ctx, from, to, amt) error
    SendCoinsFromModuleToAccount(ctx, module, to, amt) error
    SendCoinsFromAccountToModule(ctx, from, module, amt) error

    // Mint/Burn methods
    MintCoins(ctx, module, amt) error
    BurnCoins(ctx, module, amt) error
}
```

#### Extended Coin Support <a href="#extended-coin-support" id="extended-coin-support"></a>

Automatic handling of "atest" denomination:

* Converts between test and atest transparently
* Maintains fractional balances for sub-test amounts
* Ensures consistency between x/bank and x/precisebank

### Operations <a href="#operations" id="operations"></a>

#### Transfer <a href="#transfer" id="transfer"></a>

Handles both integer and fractional components:

```
// SendCoins automatically handles precision
keeper.SendCoins(ctx, from, to, sdk.NewCoins(
    sdk.NewCoin("atest", sdk.NewInt(1500000000000)), // 0.0015 test
))
```

**Algorithm:**

1. Subtract from sender (update b(sender) and f(sender))
2. Add to receiver (update b(receiver) and f(receiver))
3. Update reserve based on carry/borrow
4. Remainder unchanged (mathematical guarantee)

#### Mint <a href="#mint" id="mint"></a>

Creates new tokens with proper backing:

```
// MintCoins with extended precision
keeper.MintCoins(ctx, moduleName, sdk.NewCoins(
    sdk.NewCoin("atest", sdk.NewInt(1000000000000000000)), // 1 test
))
```

**Algorithm:**

1. Add to account (update b(account) and f(account))
2. Decrease remainder (tokens enter circulation)
3. Update reserve for consistency

#### Burn <a href="#burn" id="burn"></a>

Removes tokens from circulation:

```
// BurnCoins with extended precision
keeper.BurnCoins(ctx, moduleName, sdk.NewCoins(
    sdk.NewCoin("atest", sdk.NewInt(500000000000)),  // 0.0005 test
))
```

**Algorithm:**

1. Subtract from account (update b(account) and f(account))
2. Increase remainder (tokens leave circulation)
3. Update reserve for consistency

### Events <a href="#events" id="events"></a>

Standard bank events with extended precision amounts:

#### Transfer Events <a href="#transfer-events" id="transfer-events"></a>

| Event           | Attributes                      | Description        |
| --------------- | ------------------------------- | ------------------ |
| `transfer`      | `sender`, `recipient`, `amount` | Full atest amount  |
| `coin_spent`    | `spender`, `amount`             | Extended precision |
| `coin_received` | `receiver`, `amount`            | Extended precision |

#### Mint/Burn Events <a href="#mintburn-events" id="mintburn-events"></a>

| Event      | Attributes         | Description             |
| ---------- | ------------------ | ----------------------- |
| `coinbase` | `minter`, `amount` | Minted with 18 decimals |
| `burn`     | `burner`, `amount` | Burned with 18 decimals |

### Queries <a href="#queries" id="queries"></a>

#### gRPC <a href="#grpc" id="grpc"></a>

```
service Query {
    // Get total of all fractional balances
    rpc TotalFractionalBalances(QueryTotalFractionalBalancesRequest)
        returns (QueryTotalFractionalBalancesResponse);

    // Get current remainder amount
    rpc Remainder(QueryRemainderRequest)
        returns (QueryRemainderResponse);

    // Get fractional balance for an account
    rpc FractionalBalance(QueryFractionalBalanceRequest)
        returns (QueryFractionalBalanceResponse);
}
```

#### CLI <a href="#cli" id="cli"></a>

\`\`\`bash Query-Total # Query total fractional balances evmd query precisebank total-fractional-balances

### Example output: <a href="#example-output" id="example-output"></a>

### total: "2000000000000atest" <a href="#total-andquot2000000000000atestandquot" id="total-andquot2000000000000atestandquot"></a>

````

```bash Query-Remainder
# Query remainder amount
evmd query precisebank remainder

# Example output:
# remainder: "100atest"
````

```
# Query account fractional balance
evmd query precisebank fractional-balance Ontomir1...

# Example output:
# fractional_balance: "10000atest"
```

### Integration <a href="#integration" id="integration"></a>

#### For EVM Module <a href="#for-evm-module" id="for-evm-module"></a>

Replace bank keeper with precisebank keeper in app.go:

```
app.EvmKeeper = evmkeeper.NewKeeper(
    app.PrecisebankKeeper, // Instead of app.BankKeeper
    // ... other parameters
)
```

#### For Other Modules <a href="#for-other-modules" id="for-other-modules"></a>

Query extended balances through standard interface:

```
// Automatically handles atest denomination
balance := keeper.GetBalance(ctx, addr, "atest")

// Transfer with 18 decimal precision
err := keeper.SendCoins(ctx, from, to, 
    sdk.NewCoins(sdk.NewCoin("atest", amount)))
```

### Reserve Account <a href="#reserve-account" id="reserve-account"></a>

The reserve account maintains backing for fractional balances:

```
Reserve_test × 10^12 = Σ(all fractional balances) + remainder
```

#### Monitoring Reserve <a href="#monitoring-reserve" id="monitoring-reserve"></a>

```
# Check reserve account balance
evmd query bank balances Ontomir1m3h30wlvsf8llruxtpukdvsy0km2kum8g38c8q

# Verify invariant
evmd query precisebank total-fractional-balances
evmd query precisebank remainder
```

### Invariants <a href="#invariants" id="invariants"></a>

Critical invariants maintained by the module:

| Invariant            | Formula                                        | Description              |
| -------------------- | ---------------------------------------------- | ------------------------ |
| **Supply**           | `Total_atest = Total_test × 10^12 - remainder` | Total supply consistency |
| **Fractional Range** | `0 ≤ f(n) < 10^12`                             | Valid fractional bounds  |
| **Reserve Backing**  | `Reserve × 10^12 = Σf(n) + r`                  | Full backing guarantee   |
| **Conservation**     | `Δ(Total_atest) = Δ(Total_test × 10^12)`       | No creation/destruction  |

### Best Practices <a href="#best-practices" id="best-practices"></a>

#### Chain Integration <a href="#chain-integration" id="chain-integration"></a>

1. **Reserve Monitoring**
   * Track reserve balance for validation
   * Set up alerts for invariant violations
   * Regular audits of fractional sums
2.  **Migration Path**

    ```
    // Deploy in passive mode
    app.PrecisebankKeeper = precisebankkeeper.NewKeeper(
        app.BankKeeper,
        // Fractional balances start at zero
    )
    ```
3.  **Testing**

    ```
    // Verify fractional operations
    suite.Require().Equal(
        expectedFractional,
        keeper.GetFractionalBalance(ctx, addr),
    )
    ```

#### dApp Development <a href="#dapp-development" id="dapp-development"></a>

1.  **Balance Queries**

    ```
    // Query in atest (18 decimals)
    const balance = await queryClient.precisebank.fractionalBalance({
        address: "Ontomir1..."
    });
    ```
2.  **Precision Handling**

    ```
    // Convert between precisions
    const testAmount = atestAmount / BigInt(10**12);
    const atestAmount = testAmount * BigInt(10**12);
    ```

### Security Considerations <a href="#security-considerations" id="security-considerations"></a>

#### Overflow Protection <a href="#overflow-protection" id="overflow-protection"></a>

* All arithmetic uses checked math
* Fractional values bounded to \[0, 10^12)
* Integer overflow impossible by design

#### Atomicity <a href="#atomicity" id="atomicity"></a>

* Balance updates are atomic
* Reserve adjustments in same transaction
* No intermediate states visible

#### Precision Guarantees <a href="#precision-guarantees" id="precision-guarantees"></a>

* No precision loss during operations
* All fractional amounts preserved
* Rounding only at display layer

### Performance <a href="#performance" id="performance"></a>

#### Storage Impact <a href="#storage-impact" id="storage-impact"></a>

* Additional O(n) storage for accounts with fractional balances
* Most accounts have zero fractional balance (no storage)
* Reserve account: single additional balance

#### Computation <a href="#computation" id="computation"></a>

* Constant time operations for all transfers
* Single addition/multiplication for balance queries
* \~10% gas overhead for fractional updates

#### Optimization <a href="#optimization" id="optimization"></a>

* Lazy initialization (fractional balances start at zero)
* Sparse storage (only non-zero fractions stored)
* Batch operations maintain efficiency

### Troubleshooting <a href="#troubleshooting" id="troubleshooting"></a>

#### Common Issues <a href="#common-issues" id="common-issues"></a>

| Issue                  | Cause                | Solution                    |
| ---------------------- | -------------------- | --------------------------- |
| "fractional overflow"  | Fractional > 10^12   | Check calculation logic     |
| "insufficient balance" | Including fractional | Verify full atest balance   |
| "invariant violation"  | Supply mismatch      | Audit reserve and remainder |

#### Validation Commands <a href="#validation-commands" id="validation-commands"></a>

```
# Verify module invariants
evmd query precisebank total-fractional-balances
evmd query precisebank remainder

# Check specific account
evmd query bank balances Ontomir1... --denom test
evmd query precisebank fractional-balance Ontomir1...
```
