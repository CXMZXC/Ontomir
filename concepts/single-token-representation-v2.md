# Single Token Representation v2

> Understanding unified token models across Ontomir and EVM ecosystems

### The Problem: Token Fragmentation <a href="#the-problem-token-fragmentation" id="the-problem-token-fragmentation"></a>

In traditional blockchain ecosystems with EVM support, tokens often exist in multiple representations:

* **Native tokens** in the base blockchain format
* **Wrapped tokens** as ERC20 contracts
* **IBC vouchers** for cross-chain assets
* **Bridge tokens** from various bridge protocols

This fragmentation creates confusion, liquidity splitting, and poor user experience. Users must manually wrap/unwrap tokens, track multiple versions, and worry about using the correct representation.

### Single Token Representation v2 (STRv2) <a href="#single-token-representation-v2-strv2" id="single-token-representation-v2-strv2"></a>

STRv2 solves fragmentation by ensuring each asset has exactly one canonical representation that automatically adapts to its usage context.

#### Core Principles <a href="#core-principles" id="core-principles"></a>

**One Asset, One Representation** Each token exists in either Ontomir or EVM format at any given time, never both simultaneously. The representation automatically switches based on where it's being used.

**Automatic Conversion** When tokens move between Ontomir and EVM contexts, conversion happens transparently:

* Sending to an Ethereum address → converts to ERC20
* Sending to a Ontomir address → converts to native coin
* IBC transfer with EVM recipient → arrives as ERC20

**Native Performance** Unlike wrapped tokens that require contract calls, STRv2 tokens operate at native speed through precompiled contracts that directly access bank balances.

### Token Pairs: The Bridge <a href="#token-pairs-the-bridge" id="token-pairs-the-bridge"></a>

A TokenPair links a Ontomir denomination with an ERC20 contract address, establishing the canonical mapping:

```
Ontomir Coin (test) ←→ TokenPair ←→ ERC20 Contract (0x...)
```

#### Origin Types <a href="#origin-types" id="origin-types"></a>

**Native Ontomir Coins**

* Start as Ontomir SDK coins (test, uosmo, etc.)
* Module deploys an ERC20 contract representation
* Contract owned by module (OWNER\_MODULE)
* Conversions use mint/burn mechanism

**Existing ERC20 Tokens**

* Start as deployed ERC20 contracts
* Module creates a Ontomir coin denomination
* Contract owned externally (OWNER\_EXTERNAL)
* Conversions use escrow/release mechanism

### Precompiled Contracts: The Magic <a href="#precompiled-contracts-the-magic" id="precompiled-contracts-the-magic"></a>

Traditional wrapped tokens require expensive contract storage and logic. STRv2 uses precompiled contracts - native code that appears as ERC20 to the EVM but executes at native speed.

#### Native Precompiles <a href="#native-precompiles" id="native-precompiles"></a>

For Ontomir-native assets, precompiles provide:

* Standard ERC20 interface (transfer, approve, balanceOf)
* Direct bank module integration
* No contract storage overhead
* Gas costs 10-100x lower than regular ERC20

#### Dynamic Precompiles <a href="#dynamic-precompiles" id="dynamic-precompiles"></a>

For advanced functionality:

* WERC20 interfaces with deposit/withdraw
* Custom extensions per token
* Runtime registration
* Module-specific features

### Conversion Mechanics <a href="#conversion-mechanics" id="conversion-mechanics"></a>

#### Ontomir → ERC20 <a href="#ontomir-erc20" id="ontomir-erc20"></a>

When converting native coins to ERC20:

1. **User initiates** conversion via message or automatic trigger
2. **Module validates** token pair is enabled
3. **Coins escrowed** in module account (removed from circulation)
4. **ERC20 minted** to recipient address
5. **Event emitted** for tracking

The coins don't disappear - they're held in escrow, maintaining supply consistency.

#### ERC20 → Ontomir <a href="#erc20-ontomir" id="erc20-ontomir"></a>

When converting ERC20 to native coins:

1. **User initiates** conversion or automatic trigger
2. **Module validates** sufficient ERC20 balance
3. **Tokens burned** (if module-owned) or **escrowed** (if external)
4. **Coins released** from module account
5. **Event emitted** for tracking

### IBC Integration <a href="#ibc-integration" id="ibc-integration"></a>

STRv2 seamlessly integrates with IBC transfers through middleware:

#### Automatic Conversion <a href="#automatic-conversion" id="automatic-conversion"></a>

When receiving IBC tokens:

* If recipient is Ethereum address → convert to ERC20
* If recipient is Ontomir address → keep as native
* If token pair doesn't exist → create dynamically

#### Cross-Chain DeFi <a href="#cross-chain-defi" id="cross-chain-defi"></a>

This enables powerful workflows:

1. Receive OSMO from Osmosis via IBC
2. Automatically converts to ERC20
3. Use in Uniswap-style DEX
4. Swap for other tokens
5. Send back via IBC as native

### Design Rationale <a href="#design-rationale" id="design-rationale"></a>

#### Why Not Simple Wrapping? <a href="#why-not-simple-wrapping" id="why-not-simple-wrapping"></a>

Traditional wrapping (like WETH) has drawbacks:

* **Double tokens**: Native ETH and WETH coexist
* **Manual process**: Users must wrap/unwrap
* **Liquidity split**: Separate pools for each version
* **Gas overhead**: Extra contract calls

#### Why Precompiles? <a href="#why-precompiles" id="why-precompiles"></a>

Precompiles provide:

* **Native speed**: Direct state access
* **Compatibility**: Standard ERC20 interface
* **Efficiency**: No storage overhead
* **Transparency**: Appears as regular token to contracts

#### Why Automatic Conversion? <a href="#why-automatic-conversion" id="why-automatic-conversion"></a>

Automatic conversion improves UX:

* **No manual steps**: Tokens "just work"
* **Unified liquidity**: Single pool per asset
* **Reduced errors**: No wrong token mistakes
* **Better bridging**: Seamless cross-chain

### Security Considerations <a href="#security-considerations" id="security-considerations"></a>

#### Supply Integrity <a href="#supply-integrity" id="supply-integrity"></a>

The total supply remains constant across conversions:

```
Total Supply = Ontomir Circulating + ERC20 Circulating + Module Escrow
```

#### Ownership Models <a href="#ownership-models" id="ownership-models"></a>

**Module Ownership** (Native Coins):

* Module can mint/burn ERC20
* Ensures 1:1 backing
* No external control risks

**External Ownership** (ERC20 Origin):

* Original deployer retains control
* Module uses escrow mechanism
* Preserves contract upgradeability

#### Attack Vectors <a href="#attack-vectors" id="attack-vectors"></a>

**Malicious ERC20 Contracts**

* Validation before registration
* Event log verification
* Balance checks pre/post operation

**Reentrancy Protection**

* State changes before external calls
* Checks-effects-interactions pattern
* Gas limit enforcement

### Benefits for Users <a href="#benefits-for-users" id="benefits-for-users"></a>

#### Simplified Experience <a href="#simplified-experience" id="simplified-experience"></a>

* One token to rule them all
* No manual wrapping
* Automatic conversion where needed
* Consistent balance across contexts

#### Better Capital Efficiency <a href="#better-capital-efficiency" id="better-capital-efficiency"></a>

* No liquidity fragmentation
* Single market per asset
* Lower slippage
* Unified order books

#### Enhanced Composability <a href="#enhanced-composability" id="enhanced-composability"></a>

* Use any token in any context
* Seamless DeFi integration
* Cross-chain applications
* Future-proof design

### Benefits for Developers <a href="#benefits-for-developers" id="benefits-for-developers"></a>

#### Easier Integration <a href="#easier-integration" id="easier-integration"></a>

* Standard interfaces everywhere
* No wrapper contract management
* Automatic handling
* Less code complexity

#### Gas Optimization <a href="#gas-optimization" id="gas-optimization"></a>

* Native precompile efficiency
* No extra contract calls
* Batched operations
* Lower user costs

#### Innovation Enablement <a href="#innovation-enablement" id="innovation-enablement"></a>

* Build cross-VM applications
* Leverage both ecosystems
* New DeFi primitives
* Unified token standards

### Future Evolution <a href="#future-evolution" id="future-evolution"></a>

#### Potential Enhancements <a href="#potential-enhancements" id="potential-enhancements"></a>

**Multi-VM Support**

* CosmWasm integration
* Move VM compatibility
* Universal token representation

**Advanced Features**

* Streaming payments
* Programmable transfers
* Conditional conversions
* Batch operations

**Cross-Chain Standards**

* IBC native tokens
* Universal asset IDs
* Metadata preservation
* Governance coordination
