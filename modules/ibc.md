# IBC

> Inter-Blockchain Communication protocol implementation with EVM callbacks

The `x/ibc` module from [Ontomir/evm](https://github.com/Ontomir/evm) implements Inter-Blockchain Communication (IBC) protocol support with specialized EVM callback functionality for cross-chain smart contract interactions.

### Overview <a href="#overview" id="overview"></a>

The IBC module extends the standard IBC protocol with EVM-specific features:

* **IBC Callbacks**: Execute EVM contracts automatically during IBC packet lifecycle
* **Cross-chain Contract Calls**: Enable smart contracts to interact across chains
* **Packet Lifecycle Management**: Handle acknowledgments and timeouts through EVM contracts

### Components <a href="#components" id="components"></a>

#### IBC Callbacks <a href="#ibc-callbacks" id="ibc-callbacks"></a>

The EVM Callbacks module implements the EVM contractKeeper interface that interacts with ibc-go's [callbacks middleware](https://github.com/Ontomir/ibc-go/blob/main/modules/apps/callbacks/README.md), specifically for ICS-20 transfer applications.

**Key Features:**

* **Destination Callbacks**: Execute contracts on packet receipt (`onRecvPacket`)
* **Source Callbacks**: Handle acknowledgments (`onAcknowledgePacket`) and timeouts (`onTimeoutPacket`)
* **Atomic Execution**: Contract calls happen atomically with token transfers

#### IBC Transfer Integration <a href="#ibc-transfer-integration" id="ibc-transfer-integration"></a>

The module works closely with the ICS20 transfer application to enable:

* Cross-chain token transfers to EVM contracts
* Automatic contract execution with received funds
* Custom calldata propagation across chains

Smart contracts can initiate IBC transfers using the \[ICS20 Precompile]\(/docs/evm/next/documentation/smart-contracts/precompiles/ics20), which provides the \`transfer\` function with memo field support for callbacks. \*\*Address Format Limitation\*\*: Currently, IBC transfer receiver addresses must be in bech32 format (e.g., \`Ontomir1...\`). While sender addresses are automatically converted from hex to bech32, receiver addresses must be provided in bech32 format. Full hex address support for receivers is planned for a future release.

### Callback Types <a href="#callback-types" id="callback-types"></a>

#### Destination Callbacks (`onRecvPacket`) <a href="#destination-callbacks-onrecvpacket" id="destination-callbacks-onrecvpacket"></a>

Executed on the destination chain when a packet is received, allowing contracts to:

* Receive cross-chain tokens
* Execute custom logic with the received funds
* Perform operations like DEX swaps or liquidity provision

#### Source Callbacks (`onAcknowledgePacket` & `onTimeoutPacket`) <a href="#source-callbacks-onacknowledgepacket-andamp-ontimeoutpacket" id="source-callbacks-onacknowledgepacket-andamp-ontimeoutpacket"></a>

Executed on the source chain when packet lifecycle completes, enabling contracts to:

* Handle successful transfer acknowledgments
* Recover funds from failed/timed out transfers
* Implement retry logic for failed transfers

### Implementation Details <a href="#implementation-details" id="implementation-details"></a>

#### Memo Format <a href="#memo-format" id="memo-format"></a>

EVM callbacks use the `memo` field in ICS-20 transfers with specific JSON structure:

**Destination Callback:**

```
{
  "dest_callback": {
    "address": "0x...",
    "gas_limit": "1000000", 
    "calldata": "0x..."
  }
}
```

**Source Callback:**

```
{
  "src_callback": {
    "address": "0x...",
    "gas_limit": "1000000"
  }
}
```

#### Security Considerations <a href="#security-considerations" id="security-considerations"></a>

* **Isolated Addresses**: Destination callbacks use ephemeral addresses to prevent confusion with local accounts
* **Sender Validation**: Source callbacks validate that only the packet sender can set callbacks
* **Gas Limits**: Callback execution is bounded by specified gas limits
* IBC Overview - IBC concepts and fundamentals
* ICS20 Precompile - Cross-chain token transfers
* Callbacks Interface - Smart contract callback interface

### External Resources <a href="#external-resources" id="external-resources"></a>

* [IBC Protocol Specification](https://ibc.ontomir.network/)
* [IBC-Go Callbacks Middleware](https://github.com/Ontomir/ibc-go/blob/main/modules/apps/callbacks/README.md)
* [ICS-20 Token Transfer](https://github.com/Ontomir/ibc/tree/master/spec/app/ics-020-fungible-token-transfer)
