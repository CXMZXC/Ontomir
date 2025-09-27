# Precision Handling

> Understanding decimal precision bridging between Ontomir and EVM

### The Precision Challenge <a href="#the-precision-challenge" id="the-precision-challenge"></a>

Ontomir SDK and Ethereum use different decimal precisions for their native tokens:

* **Ontomir SDK**: 6 decimals (1 ATOM = 10^6 test)
* **Ethereum**: 18 decimals (1 ETH = 10^18 wei)

This 12-decimal difference creates challenges when bridging the two ecosystems. Simply scaling values would lose precision or create rounding errors.

### Mathematical Foundation <a href="#mathematical-foundation" id="mathematical-foundation"></a>

#### Balance Representation <a href="#balance-representation" id="balance-representation"></a>

Any account balance can be decomposed into integer and fractional components:

```
Total Balance = Integer Part × Conversion Factor + Fractional Part
```

For an account with balance `a(n)` in smallest units (18 decimals):

```
a(n) = b(n) × C + f(n)
```

Where:

* `a(n)` = Total balance in 18-decimal units (atest)
* `b(n)` = Integer balance in 6-decimal units (test)
* `f(n)` = Fractional balance (remainder)
* `C` = Conversion factor (10^12)
* Constraint: `0 ≤ f(n) < C`

#### Example Decomposition <a href="#example-decomposition" id="example-decomposition"></a>

Consider a balance of 1,234,567,890,123,456,789 atest:

```
1,234,567,890,123,456,789 = 1,234,567 × 10^12 + 890,123,456,789

Where:
- b(n) = 1,234,567 test (stored in bank)
- f(n) = 890,123,456,789 atest (stored in precisebank)
- Total = 1.234567890123456789 ATOM
```

### The Reserve Account <a href="#the-reserve-account" id="the-reserve-account"></a>

To maintain supply consistency, a reserve account holds backing for all fractional balances:

#### Reserve Equation <a href="#reserve-equation" id="reserve-equation"></a>

```
Reserve Balance × C = Sum of All Fractional Balances + Remainder
```

Formally:

```
b(R) × C = Σf(n) + r
```

Where:

* `b(R)` = Reserve balance in test
* `Σf(n)` = Sum of all account fractional balances
* `r` = Remainder (fractional amount not in circulation)
* Constraint: `0 ≤ r < C`

#### Supply Invariant <a href="#supply-invariant" id="supply-invariant"></a>

This ensures the fundamental invariant:

```
Total_atest = Total_test × 10^12 - remainder
```

The remainder represents sub-test amounts that exist in the system but aren't assigned to any specific account.

### Operation Algorithms <a href="#operation-algorithms" id="operation-algorithms"></a>

#### Transfer Algorithm <a href="#transfer-algorithm" id="transfer-algorithm"></a>

Transferring between accounts requires careful handling of carries and borrows:

**From account 1 to account 2, amount `a`:**

1.  **Calculate new fractional balances:**

    ```
    f'(1) = (f(1) - a) mod C
    f'(2) = (f(2) + a) mod C
    ```
2.  **Handle integer updates with carry/borrow:**

    ```
    b'(1) = b(1) - ⌊a/C⌋ - (f'(1) > f(1) ? 1 : 0)
    b'(2) = b(2) + ⌊a/C⌋ + (f'(2) < f(2) ? 1 : 0)
    ```
3. **Update reserve based on carry conditions:**
   * Both carry/borrow: No reserve change
   * Only sender borrows: Reserve decreases by 1
   * Only receiver carries: Reserve increases by 1

#### Mint Algorithm <a href="#mint-algorithm" id="mint-algorithm"></a>

Creating new tokens while maintaining backing:

1.  **Update account:**

    ```
    f'(n) = (f(n) + a) mod C
    b'(n) = b(n) + ⌊a/C⌋ + (f'(n) < f(n) ? 1 : 0)
    ```
2.  **Update remainder:**

    ```
    r' = (r - a) mod C
    ```
3. **Adjust reserve for consistency**

#### Burn Algorithm <a href="#burn-algorithm" id="burn-algorithm"></a>

Removing tokens from circulation:

1.  **Update account:**

    ```
    f'(n) = (f(n) - a) mod C
    b'(n) = b(n) - ⌊a/C⌋ - (f'(n) > f(n) ? 1 : 0)
    ```
2.  **Update remainder:**

    ```
    r' = (r + a) mod C
    ```
3. **Adjust reserve for consistency**

### Proof: Remainder Unchanged in Transfers <a href="#proof-remainder-unchanged-in-transfers" id="proof-remainder-unchanged-in-transfers"></a>

A critical property is that transfers don't change the global remainder:

**Proof:**

1. Start with transfer affecting fractional balances
2. Take modulo C of the balance equation
3. Since reserve changes are multiples of C, they vanish mod C
4. Fractional changes sum to zero (amount subtracted equals amount added)
5. Therefore: `r' = r`

This elegant property means transfers are purely redistributive - they don't create or destroy fractional amounts.

### Precision Hierarchy <a href="#precision-hierarchy" id="precision-hierarchy"></a>

The system manages three precision levels:

| Unit      | Precision | Decimals | Usage          |
| --------- | --------- | -------- | -------------- |
| **ATOM**  | 10^0      | 0        | Human display  |
| **test**  | 10^-6     | 6        | Ontomir native |
| **atest** | 10^-18    | 18       | EVM native     |

#### Conversion Examples <a href="#conversion-examples" id="conversion-examples"></a>

```
1 ATOM = 1,000,000 test = 1,000,000,000,000,000,000 atest
0.000001 ATOM = 1 test = 1,000,000,000,000 atest
0.000000000000000001 ATOM = 0.000000000001 test = 1 atest
```

### Edge Cases and Limits <a href="#edge-cases-and-limits" id="edge-cases-and-limits"></a>

#### Minimum Transferable Amount <a href="#minimum-transferable-amount" id="minimum-transferable-amount"></a>

* **In Ontomir**: 1 test (0.000001 ATOM)
* **In EVM**: 1 atest (0.000000000000000001 ATOM)

#### Maximum Precision <a href="#maximum-precision" id="maximum-precision"></a>

* **Ontomir operations**: Limited to test granularity
* **EVM operations**: Full atest precision
* **Cross-system**: Automatic precision handling

#### Dust Amounts <a href="#dust-amounts" id="dust-amounts"></a>

Amounts smaller than 1 test but larger than 0 atest:

* Tracked in fractional balances
* Accumulate until reaching 1 test
* Never lost or rounded away

### Implementation Strategy <a href="#implementation-strategy" id="implementation-strategy"></a>

#### Storage Optimization <a href="#storage-optimization" id="storage-optimization"></a>

Rather than storing 18-decimal balances directly:

1. Store 6-decimal amounts in existing bank module
2. Store only the fractional remainder separately
3. Reconstruct full precision on demand

This approach:

* Maintains backward compatibility
* Minimizes storage overhead
* Preserves full precision

#### Query Performance <a href="#query-performance" id="query-performance"></a>

When querying balances:

```
fullBalance = bankBalance * 10^12 + fractionalBalance
```

This single multiplication and addition reconstructs the full 18-decimal balance efficiently.

### Why This Matters <a href="#why-this-matters" id="why-this-matters"></a>

#### For Users <a href="#for-users" id="for-users"></a>

* **No precision loss**: Every atest is accounted for
* **Seamless experience**: Automatic handling across systems
* **Fair transactions**: No rounding advantages or disadvantages

#### For Developers <a href="#for-developers" id="for-developers"></a>

* **Standard interfaces**: Use familiar decimals for each system
* **Automatic conversion**: No manual precision management
* **Predictable behavior**: Mathematical guarantees on operations

#### For the Ecosystem <a href="#for-the-ecosystem" id="for-the-ecosystem"></a>

* **True interoperability**: Native precision for both ecosystems
* **Future proof**: Extensible to other precision requirements
* **Efficient design**: Minimal overhead for maximum capability

### Comparison with Alternatives <a href="#comparison-with-alternatives" id="comparison-with-alternatives"></a>

#### Simple Scaling <a href="#simple-scaling" id="simple-scaling"></a>

```
// Naive approach - loses precision
evmAmount = OntomirAmount * 10^12
```

**Problems**: Loses sub-test amounts, rounding errors accumulate

#### Fixed-Point Arithmetic <a href="#fixed-point-arithmetic" id="fixed-point-arithmetic"></a>

```
// Complex fixed-point math
amount = FixedPoint{mantissa: 123456, exponent: -18}
```

**Problems**: Complex implementation, performance overhead, compatibility issues

#### Separate Balances <a href="#separate-balances" id="separate-balances"></a>

```
// Maintain two separate balance systems
OntomirBalance: 1000000 test
evmBalance: 1000000000000000000 atest
```

**Problems**: Synchronization issues, double accounting, complexity

#### PreciseBank Solution <a href="#precisebank-solution" id="precisebank-solution"></a>

```
// Elegant decomposition
totalBalance = integerPart * 10^12 + fractionalPart
```

**Advantages**: Simple, efficient, precise, compatible

### Future Extensions <a href="#future-extensions" id="future-extensions"></a>

#### Multi-Precision Support <a href="#multi-precision-support" id="multi-precision-support"></a>

The mathematical framework extends to any precision:

* 8 decimals for certain tokens
* 27 decimals for high-precision applications
* Variable precision based on token type

#### Cross-Chain Precision <a href="#cross-chain-precision" id="cross-chain-precision"></a>

Standardized precision handling for:

* IBC transfers with different precisions
* Bridge protocols with varying decimals
* Universal precision abstraction layer
