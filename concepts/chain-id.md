# Chain ID

> Chain IDs are unique identifiers that distinguish blockchain networks from each other. Ontomir EVM uses a dual Chain ID system to maintain compatibility with both Ontomir SDK and Ethereum ecosystems.

You can look up existing EVM Chain IDs by referring to \[chainlist.org]\(https://chainlist.org/) to ensure your chosen ID is not already in use.

### Dual Chain ID System <a href="#dual-chain-id-system" id="dual-chain-id-system"></a>

Ontomir EVM requires **two separate chain IDs** to maintain full compatibility with both the Ontomir SDK and Ethereum ecosystems:

#### 1. Ontomir Chain ID (String) <a href="#id-1-ontomir-chain-id-string" id="id-1-ontomir-chain-id-string"></a>

The **Ontomir Chain ID** is a string identifier used by:

* CometBFT consensus engine
* IBC (Inter-Blockchain Communication) protocol
* Native Ontomir SDK transactions
* Chain upgrades and governance

**Format**: String with flexible naming (e.g., `"Ontomirevm-1"`, `"mychain-testnet-2"`)

**Example**:

```
// In genesis.json
{
  "chain_id": "Ontomirevm-1"
}
```

#### 2. EVM Chain ID (Integer) <a href="#id-2-evm-chain-id-integer" id="id-2-evm-chain-id-integer"></a>

The **EVM Chain ID** is an integer used by:

* Ethereum transactions (EIP-155 replay protection)
* MetaMask and other Ethereum wallets
* Smart contract deployments
* EVM tooling (Hardhat, Foundry, etc.)

**Format**: Positive integer (e.g., `9000`, `9001`)

**Example**:

```
// In app/app.go
const EVMChainID = 9000
```

### Configuration <a href="#configuration" id="configuration"></a>

Both chain IDs must be configured when setting up your chain:

#### In Your Application Code <a href="#in-your-application-code" id="in-your-application-code"></a>

```
// app/app.go
const (
    OntomirChainID = "Ontomirevm-1"  // String for Ontomir/IBC
    EVMChainID    = 9000           // Integer for EVM/Ethereum
)
```

#### In Genesis Configuration <a href="#in-genesis-configuration" id="in-genesis-configuration"></a>

The Ontomir Chain ID is set in `genesis.json`:

```
{
  "chain_id": "Ontomirevm-1",
  // ... other genesis parameters
}
```

The EVM Chain ID is configured in the EVM module parameters:

```
// During chain initialization
evmtypes.DefaultChainConfig(9000)  // Your EVM Chain ID
```

### Important Considerations <a href="#important-considerations" id="important-considerations"></a>

#### Chain ID Selection <a href="#chain-id-selection" id="chain-id-selection"></a>

The two chain IDs serve different purposes and \*\*must not be confused\*\*:

* Use the **Ontomir Chain ID** (string) for IBC connections, governance proposals, and Ontomir SDK operations
* Use the **EVM Chain ID** (integer) for MetaMask configuration, smart contract deployments, and EIP-155 transaction signing

#### EVM Chain ID Guidelines <a href="#evm-chain-id-guidelines" id="evm-chain-id-guidelines"></a>

When selecting your EVM Chain ID:

1. **Check availability**: Verify your chosen ID is not already in use on [chainlist.org](https://chainlist.org/)
2. **Avoid conflicts**: Don't use well-known chain IDs (1 for Ethereum mainnet, 137 for Polygon, etc.)
3. **Consider ranges**:
   * Production networks often use lower numbers
   * Testnets commonly use higher ranges (9000+)
   * Local development can use very high numbers (31337+)

#### Chain Upgrades <a href="#chain-upgrades" id="chain-upgrades"></a>

Unlike traditional Ontomir chains that change their chain ID during upgrades (e.g., \`Ontomirhub-4\` to \`Ontomirhub-5\`), the EVM Chain ID must remain \*\*constant\*\* across upgrades to maintain compatibility with deployed smart contracts and existing wallets.

Only the Ontomir Chain ID may change during chain upgrades if needed for consensus-breaking changes. The EVM Chain ID should never change once set.

### Examples <a href="#examples" id="examples"></a>

Here are some example configurations:

| Network Type | Ontomir Chain ID         | EVM Chain ID | Notes               |
| ------------ | ------------------------ | ------------ | ------------------- |
| Mainnet      | `"Ontomirevm-1"`         | `9000`       | Production network  |
| Testnet      | `"Ontomirevm-testnet-1"` | `9001`       | Public testnet      |
| Devnet       | `"Ontomirevm-dev-1"`     | `9002`       | Development network |
| Local        | `"Ontomirevm-local-1"`   | `31337`      | Local development   |

### Troubleshooting <a href="#troubleshooting" id="troubleshooting"></a>

#### Common Issues <a href="#common-issues" id="common-issues"></a>

1. **"Chain ID mismatch" errors**
   * **Cause**: Using Ontomir Chain ID where EVM Chain ID is expected (or vice versa)
   * **Solution**: Ensure you're using the correct type of chain ID for each context
2. **MetaMask connection failures**
   * **Cause**: Incorrect EVM Chain ID in wallet configuration
   * **Solution**: Use the integer EVM Chain ID, not the string Ontomir Chain ID
3. **IBC transfer failures**
   * **Cause**: Using EVM Chain ID for IBC operations
   * **Solution**: IBC always uses the Ontomir Chain ID (string format)
4. **Smart contract deployment issues**
   * **Cause**: EIP-155 replay protection using wrong chain ID
   * **Solution**: Ensure your EVM Chain ID matches what's configured in the chain

#### Verification Commands <a href="#verification-commands" id="verification-commands"></a>

To verify your chain IDs are correctly configured:

```
# Check Ontomir Chain ID
curl -s http://localhost:26657/status | jq '.result.node_info.network'

# Check EVM Chain ID
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  http://localhost:8545 | jq '.result'
```
