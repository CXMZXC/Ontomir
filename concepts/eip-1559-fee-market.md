# EIP-1559 Fee Market

> Understanding dynamic fee pricing and the EIP-1559 mechanism in Ontomir EVM chains

### Overview <a href="#overview" id="overview"></a>

[EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) revolutionizes transaction fee mechanics by replacing the single gas price auction model with a dual-fee structure. This creates more predictable fees and improved network efficiency for EVM-compatible chains.

### Core Concepts <a href="#core-concepts" id="core-concepts"></a>

#### Dual-Fee Structure <a href="#dual-fee-structure" id="dual-fee-structure"></a>

Before EIP-1559, fees were simple:

```
fee = gasPrice * gasLimit
```

With EIP-1559, fees have two components:

```
fee = (baseFee + priorityTip) * gasLimit
```

where:

* `baseFee`: Protocol-determined minimum price per gas unit
* `priorityTip`: Optional fee for transaction prioritization

The Ontomir SDK uses different terminology than Ethereum. What Ethereum calls \`gasLimit\` is \`gasWanted\` in Ontomir. You'll encounter both terms in the Ontomir EVM as it bridges Ethereum and Ontomir SDK concepts.

#### Base Fee Mechanism <a href="#base-fee-mechanism" id="base-fee-mechanism"></a>

The base fee is the minimum price per unit of gas required for transaction inclusion. It automatically adjusts each block based on network utilization.

**Adjustment Formula**

```
NewBaseFee = BaseFee × (1 + (|GasUsed - GasTarget| / GasTarget / Denominator))
```

The adjustment direction depends on block utilization:

* **Increases** when `gasUsed > gasTarget` (congested)
* **Decreases** when `gasUsed < gasTarget` (underutilized)
* **Remains stable** when `gasUsed == gasTarget` (optimal)

**Example Calculation**

With standard Ethereum parameters:

* Current base fee: 1000 units
* Block capacity: 10M gas
* Target (50%): 5M gas
* Actual usage: 8M gas (80% full)
* Denominator: 8

```
Adjustment = (8M - 5M) / 5M / 8 = 0.075 (7.5% increase)
New base fee = 1000 × 1.075 = 1075 units
```

#### Target Utilization <a href="#target-utilization" id="target-utilization"></a>

The target utilization determines the optimal block fullness:

```
Target Gas = MaxBlockGas / ElasticityMultiplier
Target Utilization % = 100 / ElasticityMultiplier
```

Common configurations:

* `ElasticityMultiplier = 2`: 50% target (Ethereum standard)
* `ElasticityMultiplier = 4`: 25% target (aggressive pricing)
* `ElasticityMultiplier = 1`: 100% target (maximum throughput)

#### Priority Tips and MEV <a href="#priority-tips-and-mev" id="priority-tips-and-mev"></a>

The `max_priority_fee_per_gas` (tip) serves as an incentive for validators to include transactions faster. However, in Ontomir SDK implementations:

* Transaction prioritization is limited compared to Ethereum
* Tips may have minimal effect on inclusion order
* MEV opportunities are generally reduced

#### Effective Gas Price <a href="#effective-gas-price" id="effective-gas-price"></a>

For EIP-1559 transactions, users specify two limits:

* `maxFeePerGas`: Maximum total they're willing to pay
* `maxPriorityFeePerGas`: Maximum tip for validators

The effective price becomes:

```
effectiveGasPrice = min(baseFee + maxPriorityFeePerGas, maxFeePerGas)
```

This ensures users never pay more than their specified maximum while allowing for priority fees when network conditions permit.

### Fee Calculation Examples <a href="#fee-calculation-examples" id="fee-calculation-examples"></a>

#### Low Activity Period <a href="#low-activity-period" id="low-activity-period"></a>

* Base fee: 100 gwei
* User sets: maxFeePerGas = 200 gwei, maxPriorityFeePerGas = 2 gwei
* Effective price: 100 + 2 = 102 gwei
* User pays: 102 gwei (well below their 200 gwei max)

#### High Congestion <a href="#high-congestion" id="high-congestion"></a>

* Base fee: 180 gwei
* User sets: maxFeePerGas = 200 gwei, maxPriorityFeePerGas = 30 gwei
* Effective price: min(180 + 30, 200) = 200 gwei
* User pays: 200 gwei (capped at their maximum)

### Minimum Gas Prices <a href="#minimum-gas-prices" id="minimum-gas-prices"></a>

The fee market enforces multiple price floors:

#### Local vs. Global Minimums <a href="#local-vs-global-minimums" id="local-vs-global-minimums"></a>

1. **Local minimum**: Set by individual validators in node configuration
2. **Global minimum**: Set as the `MinGasPrice` parameter via governance
3. **Base fee**: Current protocol-calculated minimum

The effective minimum is always the highest of these three values.

If the base fee falls below the global \`MinGasPrice\`, it's automatically raised to \`MinGasPrice\`. This prevents the base fee from dropping below the spam prevention threshold.

### Benefits of EIP-1559 <a href="#benefits-of-eip-1559" id="benefits-of-eip-1559"></a>

#### For Users <a href="#for-users" id="for-users"></a>

* **Predictable fees**: Base fee is known before submitting
* **Better UX**: Wallets can reliably estimate costs
* **Fair pricing**: All users in a block pay similar rates
* **Protection**: MaxFeePerGas prevents overpayment

#### For Networks <a href="#for-networks" id="for-networks"></a>

* **Automatic adjustment**: No manual intervention needed
* **Spam resistance**: Dynamic fees deter attacks
* **Optimal utilization**: Targets efficient block usage
* **Economic stability**: Predictable fee revenue

#### For Developers <a href="#for-developers" id="for-developers"></a>

* **Simplified estimation**: Base fee is protocol-provided
* **Better tools**: Standard APIs for fee data
* **Consistent behavior**: Across EVM-compatible chains

### Common Misconceptions <a href="#common-misconceptions" id="common-misconceptions"></a>

#### "Tips Always Speed Up Transactions" <a href="#andquottips-always-speed-up-transactionsandquot" id="andquottips-always-speed-up-transactionsandquot"></a>

In Ontomir SDK chains, transaction ordering is often FIFO within the mempool, making tips less effective than on Ethereum mainnet.

#### "Base Fee Always Burns" <a href="#andquotbase-fee-always-burnsandquot" id="andquotbase-fee-always-burnsandquot"></a>

While Ethereum burns the base fee, Ontomir EVM chains may distribute it differently based on their economic model.

#### "EIP-1559 Eliminates Fee Spikes" <a href="#andquoteip-1559-eliminates-fee-spikesandquot" id="andquoteip-1559-eliminates-fee-spikesandquot"></a>

It smooths volatility but can't prevent spikes during extreme congestion. The adjustment rate is intentionally limited.

### Further Reading <a href="#further-reading" id="further-reading"></a>

* [Original EIP-1559 Proposal](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)
* [Ethereum Gas Tracker](https://etherscan.io/gastracker)
* Understanding Gas and Fees
