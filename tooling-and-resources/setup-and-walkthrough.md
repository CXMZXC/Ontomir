# Setup & Walkthrough

> An introduction to Foundry and using the `cast` function, replacing complex scripts that may otherwise require multiple rpc calls and other functions in between with a single command.

Foundry is a fast and modular toolkit for Ethereum application development written in Rust. It provides four main command-line tools—`forge`, `cast`, `anvil`, and `chisel`—to streamline everything from project setup to live-chain interactions.

Among these, `cast` serves as your "Swiss-army knife" for JSON-RPC calls, letting you execute tedious day-to-day tasks (like sending raw transactions, querying balances, fetching blocks, and looking up chain metadata) with simple, consistent commands instead of verbose `curl` scripts.

This guide walks you through installing Foundry, configuring your environment, and using the most common `cast` commands to accelerate your development workflow.

### Installation & Setup <a href="#installation-andamp-setup" id="installation-andamp-setup"></a>

#### Install Foundry <a href="#install-foundry" id="install-foundry"></a>

Run the following command to install `foundryup`, the Foundry toolchain installer. This will give you the four main binaries.

```
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

After installation, you can switch to the nightly builds for the latest features, though this is optional.

```
foundryup -i nightly
```

#### Verify Installation <a href="#verify-installation" id="verify-installation"></a>

Check that each binary was installed correctly by running:

```
foundryup --version
cast --version
forge --version
anvil --version
```

You should see version outputs for each tool (e.g., `cast 0.2.0`).

### Configuration <a href="#configuration" id="configuration"></a>

To avoid passing the `--rpc-url` flag with every command, you can set defaults in a `foundry.toml` file. This file can be placed in your project's root directory for project-specific settings, or you can create a global configuration at `~/.foundry/foundry.toml` for settings you use across all projects.

Here is an example of a global configuration for connecting to a local Ontomir EVM node:

```
# ~/.foundry/foundry.toml

# Declare your named endpoints (optional; for forge test / forking)
[rpc_endpoints]
Ontomir = "<evm_rpc_url>"

# Configure the default profile for cast/forge/anvil
[profile.default]
# Tell Foundry which chain you're targeting:
chain_id     = 4321
# Directly tell cast to use your endpoint:
eth_rpc_url  = "<evm_rpc_url>"
# Set EVM version for Solidity compilation
evm_version  = "istanbul"
```

With this global configuration, Foundry tools will automatically connect to your local node without needing additional flags.

### Basic Usage of `cast` <a href="#basic-usage-of-cast" id="basic-usage-of-cast"></a>

`cast` replaces the need for custom shell scripts and verbose `curl` commands by offering first-class RPC commands.

#### Chain Metadata <a href="#chain-metadata" id="chain-metadata"></a>

*   **Get Chain ID**

    ```
    cast chain-id --rpc-url local
    ```

    Returns the chain ID in decimal format.
*   **Get Client Version**

    ```
    cast client --rpc-url local
    ```

    Displays the client implementation and version (e.g., `reth/v1.0.0`).

### `cast` Command Reference <a href="#cast-command-reference" id="cast-command-reference"></a>

#### Chain Commands <a href="#chain-commands" id="chain-commands"></a>

| Command         | Purpose                                      |
| --------------- | -------------------------------------------- |
| `cast chain`    | Show the symbolic name of the current chain. |
| `cast chain-id` | Fetch the numeric chain ID.                  |
| `cast client`   | Get the JSON-RPC client version.             |

**Example**:

```
# Assuming FOUNDRY_RPC_URL is set or using a configured alias
cast chain
cast chain-id
cast client
```

#### Transaction Commands <a href="#transaction-commands" id="transaction-commands"></a>

| Command        | Purpose                                         |
| -------------- | ----------------------------------------------- |
| `cast send`    | Sign and publish a transaction.                 |
| `cast publish` | Publish a raw, signed transaction.              |
| `cast receipt` | Fetch the receipt for a transaction hash.       |
| `cast tx`      | Query transaction details (status, logs, etc.). |
| `cast rpc`     | Invoke any raw JSON-RPC method.                 |

*   **Sign and Send** (using a private key or local keystore):

    ```
    cast send 0xRecipientAddress "transfer(address,uint256)" "0xSomeOtherAddress,100" --private-key $YOUR_PK
    ```
*   **Publish Raw Transaction** (hex-encoded):

    ```
    cast publish 0xf86b808504a817c80082520894...
    ```
*   **Generic RPC Call**:

    ```
    cast rpc eth_sendRawTransaction 0xf86b8085...
    ```

#### Account & Block Commands <a href="#account-andamp-block-commands" id="account-andamp-block-commands"></a>

| Command             | Purpose                                            |
| ------------------- | -------------------------------------------------- |
| `cast balance`      | Get an account's balance (supports ENS and units). |
| `cast block`        | Fetch block information by number or hash.         |
| `cast block-number` | Get the latest block number.                       |
| `cast logs`         | Query event logs.                                  |

*   **Get Balance** (with unit flag):

    ```
    # Using an ENS name on Ethereum mainnet
    cast balance vitalik.eth --ether --rpc-url https://eth.merkle.io
    ```
*   **Get Block Details**:

    ```
    cast block latest
    ```

#### Utility & Conversion Commands <a href="#utility-andamp-conversion-commands" id="utility-andamp-conversion-commands"></a>

| Command                | Purpose                                            |
| ---------------------- | -------------------------------------------------- |
| `cast estimate`        | Estimate gas for a call or deployment.             |
| `cast find-block`      | Find a block by its timestamp.                     |
| `cast compute-address` | Calculate a CREATE2 address.                       |
| `cast from-bin`        | Decode binary data to hex.                         |
| `cast from-wei`        | Convert wei to a more readable unit (e.g., ether). |
| `cast to-wei`          | Convert a unit (e.g., ether) to wei.               |
| `cast abi-encode`      | Encode function arguments.                         |
| `cast keccak`          | Hash data with Keccak-256.                         |

These commands allow you to drop one-off scripts for common conversions and estimations.

### Streamlining Repetative / Tedious tasks <a href="#streamlining-repetative--tedious-tasks" id="streamlining-repetative--tedious-tasks"></a>

When performing testing or development roles you may find yourself doing one or more small but very tedious tasks frequently. In your shell's configuration file (`~/.bashrc`, `~/.zshrc`), you can alias `cast` with your default RPC URL (or multiple on various chains) to save time.

```
# Set your default RPC endpoint
export CDEV_RPC="<evm_rpc_url>"
# Create an alias
alias ccast='cast --rpc-url $CDEV_RPC'

# Now you can run commands like:
ccast balance 0x...
```

This is an extremely basic example of how you can improve your workflows by building upon the heavy lifting Foundry already does out of the box.
