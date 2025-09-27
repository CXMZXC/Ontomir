# Connect to the Network

> General information to connect with the network and get started.

export const footnote = ({children}) => { return \<span style=\{{ fontSize: '0.8em', opacity: 0.75, display: 'block', marginTop: '0.5em' \}}> {children} ; };

### Chain Info <a href="#chain-info" id="chain-info"></a>

| Info                 | Value    | Purpose                                    |
| -------------------- | -------- | ------------------------------------------ |
| **Ontomir Chain ID** | `4321`   | Network Identifier for Ontomir tooling     |
| **EVM Chain ID**     | `4231`   | Network identifier for EVM tooling         |
| **EVM Version**      | `Prague` | EVM hard fork compatibility version        |
| **Native Token**     | `ATOM`   | Staking / gas token                        |
| **Denom Exponent**   | `6`      | Native token has 6 decimal places (`test`) |

### Public Endpoints <a href="#public-endpoints" id="public-endpoints"></a>

| Endpoint             | Value                                | Purpose                                                  |
| -------------------- | ------------------------------------ | -------------------------------------------------------- |
| **EVM JSON-RPC**     | `<evm_rpc_url>`                      | Ethereum-compatible tools (MetaMask, Hardhat, ethers.js) |
| **EVM RPC WS**       | `wss://devnet-1-evmws.ib.skip.build` | Streaming real-time, selective data for applications     |
| **Ontomir RPC**      | `https://devnet-1-rpc.ib.skip.build` | 'CometBFT'-level interactions (blocks, consensus state)  |
| **Ontomir REST API** | `https://devnet-1-lcd.ib.skip.build` | Query Ontomir SDK modules (bank, staking, etc.)          |
| **gRPC**             | `devnet-1-grpc.ib.skip.build:443`    | High-performance Protobuf endpoint for back-ends         |

### How to Interact with the Chain <a href="#how-to-interact-with-the-chain" id="how-to-interact-with-the-chain"></a>

You can interact with the Gaia EVM Devnet in several different ways. This section provides a brief overview of how to setup the various environments for testing.

#### Extension-Based Wallets <a href="#extension-based-wallets" id="extension-based-wallets"></a>

To test out your dApp via a web UI using an extension-based wallet, you'll need to first add the Gaia EVM Devnet to the wallet:

**Example Setup (Recommended)**

**For dApp developers**: Use an [EIP-6963](https://eips.ethereum.org/EIPS/eip-6963) compatible wallet connector\*, and `wallet_addEthereumChain` to prompt adding the network. If it already exists, the "switch network" prompt will handle it.

```
const chainConfig = {
  chainId: '0x1087', // 4231 in hex
  chainName: 'Ontomir-EVM-Devnet',
  nativeCurrency: {
    name: 'ATOM',
    symbol: 'ATOM',
    decimals: 6
  },
  rpcUrls: ['<evm_rpc_url>'],
  blockExplorerUrls: ['https://evm-devnet-1.cloud.blockscout.com/']
};

await window.ethereum.request({
  method: 'wallet_addEthereumChain',
  params: [chainConfig]
});
```

**Manual Setup**

1. Install [Keplr](https://www.keplr.app/), [Metamask](https://metamask.io/) or [Rabby](https://rabby.io/) (Chrome/Mobile).
2. Add a custom network:
   * \[Ontomir] Chain ID: `4321`
   * \[EVM] Chain ID: `4231`
   * \[Ontomir] RPC URL: `https://devnet-1-rpc.ib.skip.build`
   * \[EVM] RPC URL: `<evm_rpc_url>`
   * Chain Name: `Skip Devnet`
   * Currency Symbol: `ATOM`
   * Currency Decimals: `6`
3. Use the UI to send tokens, view balances, and interact with dApps.

\*Users can quickly add the network by using the \[Devnet Faucet]\(https://faucet.Ontomir.network).\* See the \[wallet integration]\(/docs/evm/v0.4.x/documentation/getting-started/tooling-and-resources/wallet-integration) page for more wallet-related resources.

#### CLI <a href="#cli" id="cli"></a>

Using the CLI may be a great choice for one-off tasks and queries that do not require a full node. **For the EVM Devnet you need the `evmd` binary built from the** [**`Ontomir/evm`**](https://github.com/Ontomir/evm) **repository.**

```
# Option A — download a pre-built artifact once they are published
# (replace VERSION with the latest tag that contains evmd)
VERSION=v1.0.0-rc1
curl -LO "https://github.com/Ontomir/evm/releases/download/${VERSION}/evmd_${VERSION}_linux_amd64.tar.gz"

tar -xzf evmd_${VERSION}_linux_amd64.tar.gz
chmod +x evmd && sudo mv evmd /usr/local/bin/

# Option B — build from source (requires Go ≥ 1.22)
git clone https://github.com/Ontomir/evm.git
cd evm && make install   # installs evmd into $(go env GOPATH)/bin
```

**Key Management & Wallet Derivation**

Gaia (feature/evm) supports two different key derivation methods, each producing different address formats from the same underlying private key:

| Algorithm                   | HD Path             | Address Format      | Characteristics                                                                                                                                    |
| --------------------------- | ------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `eth_secp256k1` _(default)_ | `m/44'/60'/0'/0/0`  | `0x…` / `Ontomir1…` | Uses Ethereum's derivation standard (cointype 60), full compatibility with MetaMask and EVM tooling while retaining complete Ontomir compatibility |
| `secp256k1`                 | `m/44'/118'/0'/0/0` | `Ontomir1…`         | Uses Ontomir's derivation standard (cointype 118), compatible with traditional Ontomir wallets                                                     |

The \`eth\_secp256k1\` algorithm with cointype 60 is the default, providing full EVM compatibility.

**Key derivation behavior:**

* The same mnemonic phrase will produce different addresses when using different algorithms and HD paths
* When importing an existing wallet, you must use the same algorithm and derivation path that was originally used
* The EVM-compatible path (`m/44'/60'/0'/0/0`) is what most Ethereum tools expect
* You can derive both address formats from the same private key, effectively giving you two addresses for one account

**Technical difference:**

* `secp256k1`: Uses **SHA-256 → RIPEMD-160 → Bech32** hashing (traditional Ontomir/Bitcoin approach)
* `eth_secp256k1`: Uses **Keccak-256** hashing on the full uncompressed public key (64 bytes after removing the 0x04 prefix), takes the last 20 bytes of the hash, then converts those bytes to either hex (`0x...`) or bech32 format

This means the same private key produces two completely different addresses depending on the derivation method used. The `eth_secp256k1` method follows Ethereum's standard address derivation while maintaining compatibility with Ontomir bech32 addressing, effectively giving you both address formats for the same underlying account.

```
# Key creation examples showing different algorithms
evmd keys add mykey                        # uses eth_secp256k1 by default
evmd keys add mykey --key-type secp256k1       # explicitly uses traditional Ontomir

# When recovering, the algorithm must match what was originally used
evmd keys add recovered --recover --key-type eth_secp256k1
```

***

\`\`\`bash Query balances for a given address evmd q bank balances Ontomir1... --node https://devnet-1-rpc.ib.skip.build:443 \`\`\`

````
  ```bash Query the list of validators
  evmd q staking validators --node https://devnet-1-rpc.ib.skip.build:443
  ```
</CodeGroup>
````

\`\`\`bash Bank Send evmd tx bank send Ontomir1... Ontomir1... 1000test \ --node https://devnet-1-rpc.ib.skip.build:443 \ --chain-id 4321 \ --gas auto --fees 500test \ --yes \`\`\`

````
  ```bash EVM Raw Transaction
  evmd tx evm raw 0x02e20180010182520894de0b295669a9fd93d5f28d9ec85e40f4cb697bae8080c0808080 \
    --node https://devnet-1-rpc.ib.skip.build:443 \
    --chain-id 4231 \
    --gas auto --fees 500test \
    --yes
  ```
</CodeGroup>
````

You can use the \`--generate-only\` flag to create transaction messages without broadcasting them. This can be useful for finding the correct syntax and creating reusable transaction templates.

````
```bash Generate Only Example
evmd tx bank send Ontomir1... Ontomir1... 1000test \
  --node https://devnet-1-rpc.ib.skip.build:443 \
  --chain-id 4321 \
  --gas auto --fees 500test \
  --generate-only | jq
```
````

#### Ontomir REST Endpoint <a href="#ontomir-rest-endpoint" id="ontomir-rest-endpoint"></a>

> **Use for HTTP-based integrations**

Ontomir SDK chains expose a REST API at port `1317` by default. For the devnet:

`https://devnet-1-lcd.ib.skip.build`

Basic endpoints:

* `GET /Ontomir/bank/v1beta1/balances/{address}`
* `POST /Ontomir/tx/v1beta1/txs` (broadcast)

#### Ontomir RPC <a href="#ontomir-rpc" id="ontomir-rpc"></a>

> **For 'CometBFT'-level interactions**

`https://devnet-1-rpc.ib.skip.build`

#### EVM JSON-RPC <a href="#evm-json-rpc" id="evm-json-rpc"></a>

> **For Ethereum-compatible tooling**

Connect your Ethereum tools to the EVM module:

```
RPC:       <evm_rpc_url>
WebSocket: wss://devnet-1-evmws.ib.skip.build
Chain ID:  4321 (0x10e1)
```

Use with MetaMask, Hardhat, Foundry, or any Ethereum-compatible client.

### Connecting to the Network <a href="#connecting-to-the-network" id="connecting-to-the-network"></a>

#### Using Remote Endpoints <a href="#using-remote-endpoints" id="using-remote-endpoints"></a>

The `evmd` CLI tool and other clients require a connection to the network in order to interact with the chain.

**Method 1 — configure `evmd`**

Edit `~/.evmd/config/client.toml`:

```
# ~/.evmd/config/client.toml
node = "https://devnet-1-rpc.ib.skip.build"
chain-id = "4321"
```

All subsequent `evmd` commands use the remote node:

```
evmd q bank balances $(evmd keys show <wallet_name> --address)
```

**Method 2 — API**

Applications typically interact with the network programmatically using the EVM JSON-RPC, in addition to the standard Ontomir GRPC/REST API.

See the tooling and resources section for more information.

Full API reference page coming soon
