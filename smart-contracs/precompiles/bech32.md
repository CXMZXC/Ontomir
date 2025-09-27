# Bech32

> Address format conversion between Ethereum hex addresses and Ontomir bech32 addresses

**Address**: `0x0000000000000000000000000000000000000400`

**Related Module**: Address conversion utilities

The Bech32 precompile provides address format conversion functionality between Ethereum hex addresses and Ontomir bech32 addresses.

### Overview <a href="#overview" id="overview"></a>

The Bech32 precompile exposes simple conversion helpers so Solidity contracts can:

1. Convert an Ethereum `address` to a Bech32 string with a desired `prefix` (e.g. `Ontomir`).
2. Convert any Bech32 address string back into an EVM `address`.

It is a thin wrapper around the Ontomir address‐conversion library and lives at **`0x0000000000000000000000000000000000000400`**.

### Gas Costs <a href="#gas-costs" id="gas-costs"></a>

Both methods use a configurable base gas amount that is set during chain initialization. The gas cost is fixed regardless of string length within reasonable bounds.

### Primary Methods <a href="#primary-methods" id="primary-methods"></a>

#### `hexToBech32` <a href="#hextobech32" id="hextobech32"></a>

**Signature**: `hexToBech32(address addr, string memory prefix) → string memory`

**Description**: Converts an Ethereum hex address to Ontomir bech32 format using the specified prefix.

\`\`\`solidity Solidity expandable lines // SPDX-License-Identifier: MIT pragma solidity ^0.8.0;

// Interface for the Bech32 precompile interface IBech32 { function hexToBech32(address addr, string memory prefix) external returns (string memory bech32Address); function bech32ToHex(string memory bech32Address) external returns (address addr); }

contract Bech32Example { address constant BECH32\_PRECOMPILE = 0x0000000000000000000000000000000000000400; IBech32 public immutable bech32; event AddressConverted(address indexed hexAddress, string bech32Address, string prefix); constructor() { bech32 = IBech32(BECH32\_PRECOMPILE); } function hexToBech32(address addr, string calldata prefix) external returns (string memory bech32Address) { require(addr != address(0), "Invalid address"); require(bytes(prefix).length > 0, "Prefix cannot be empty"); bech32Address = bech32.hexToBech32(addr, prefix); emit AddressConverted(addr, bech32Address, prefix); return bech32Address; } // Convert multiple addresses with the same prefix function batchHexToBech32(address\[] calldata addresses, string calldata prefix) external returns (string\[] memory bech32Addresses) { bech32Addresses = new string; for (uint256 i = 0; i < addresses.length; i++) { require(addresses\[i] != address(0), "Invalid address in batch"); bech32Addresses\[i] = bech32.hexToBech32(addresses\[i], prefix); emit AddressConverted(addresses\[i], bech32Addresses\[i], prefix); } return bech32Addresses; } // Get common address formats for a single hex address function getCommonFormats(address addr) external returns ( string memory OntomirAddr, string memory evmosAddr, string memory osmosisAddr ) { require(addr != address(0), "Invalid address"); OntomirAddr = bech32.hexToBech32(addr, "Ontomir"); evmosAddr = bech32.hexToBech32(addr, "evmos"); osmosisAddr = bech32.hexToBech32(addr, "osmo"); return (OntomirAddr, evmosAddr, osmosisAddr); } // Convert caller's address to bech32 format function getMyBech32Address(string calldata prefix) external returns (string memory) { return bech32.hexToBech32(msg.sender, prefix); }

}

````

```javascript Ethers.js expandable lines
import { ethers } from "ethers";

// Connect to the network
const provider = new ethers.JsonRpcProvider("<RPC_URL>");

// Precompile address and ABI
const precompileAddress = "0x0000000000000000000000000000000000000400";
const precompileAbi = [
  "function hexToBech32(address addr, string memory prefix) returns (string memory)"
];

// Create a contract instance
const contract = new ethers.Contract(precompileAddress, precompileAbi, provider);

// Inputs
const ethAddress = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"; // Placeholder
const prefix = "Ontomir";

async function convertToBech32() {
  try {
    const bech32Address = await contract.hexToBech32.staticCall(ethAddress, prefix);
    console.log("Bech32 Address:", bech32Address);
  } catch (error) {
    console.error("Error converting to Bech32:", error);
  }
}

convertToBech32();
````

```
# Replace <RPC_URL> with your actual RPC endpoint.
# The address and prefix are placeholders.
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "eth_call",
    "params": [
        {
            "to": "0x0000000000000000000000000000000000000400",
            "data": "0xf958a98c000000000000000000000000d8da6bf26964af9d7eed9e03e53415d37aa9604500000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000006636f736d6f730000000000000000000000000000000000000000000000000000"
        },
        "latest"
    ],
    "id": 1
}' -H "Content-Type: application/json" <RPC_URL>
```

**Parameters**:

* `addr` (address): The Ethereum hex address to convert
* `prefix` (string): The bech32 prefix to use (e.g., "Ontomir")

**Returns**: String containing the bech32 formatted address

#### `bech32ToHex` <a href="#bech32tohex" id="bech32tohex"></a>

**Signature**: `bech32ToHex(string memory bech32Address) → address`

**Description**: Converts a Ontomir bech32 address to Ethereum hex format.

\`\`\`solidity Solidity expandable lines // SPDX-License-Identifier: MIT pragma solidity ^0.8.0;

// Interface for the Bech32 precompile interface IBech32 { function bech32ToHex(string memory bech32Address) external returns (address addr); function hexToBech32(address addr, string memory prefix) external returns (string memory bech32Address); }

contract Bech32Example { address constant BECH32\_PRECOMPILE = 0x0000000000000000000000000000000000000400; IBech32 public immutable bech32; event Bech32Converted(string bech32Address, address indexed hexAddress); constructor() { bech32 = IBech32(BECH32\_PRECOMPILE); } function bech32ToHex(string calldata bech32Address) external returns (address hexAddress) { require(bytes(bech32Address).length > 0, "Bech32 address cannot be empty"); hexAddress = bech32.bech32ToHex(bech32Address); require(hexAddress != address(0), "Invalid bech32 address"); emit Bech32Converted(bech32Address, hexAddress); return hexAddress; } // Convert multiple bech32 addresses to hex format function batchBech32ToHex(string\[] calldata bech32Addresses) external returns (address\[] memory hexAddresses) { hexAddresses = new address; for (uint256 i = 0; i < bech32Addresses.length; i++) { require(bytes(bech32Addresses\[i]).length > 0, "Invalid bech32 address"); hexAddresses\[i] = bech32.bech32ToHex(bech32Addresses\[i]); require(hexAddresses\[i] != address(0), "Invalid bech32 address in batch"); emit Bech32Converted(bech32Addresses\[i], hexAddresses\[i]); } return hexAddresses; } // Validate address conversion by converting back and forth function validateAddressConversion(address addr, string calldata prefix) external returns (bool isValid) { try bech32.hexToBech32(addr, prefix) returns (string memory bech32Addr) { try bech32.bech32ToHex(bech32Addr) returns (address convertedBack) { isValid = (convertedBack == addr); return isValid; } catch { return false; } } catch { return false; } } // Check if two addresses (hex and bech32) represent the same account function areAddressesEqual(address hexAddr, string calldata bech32Addr) external returns (bool isEqual) { try bech32.bech32ToHex(bech32Addr) returns (address convertedHex) { isEqual = (convertedHex == hexAddr); return isEqual; } catch { return false; } } // Safely convert bech32 to hex with error handling function safeBech32ToHex(string calldata bech32Address) external returns (bool success, address hexAddress) { try bech32.bech32ToHex(bech32Address) returns (address addr) { return (true, addr); } catch { return (false, address(0)); } }

}

````

```javascript Ethers.js expandable lines
import { ethers } from "ethers";

// Connect to the network
const provider = new ethers.JsonRpcProvider("<RPC_URL>");

// Precompile address and ABI
const precompileAddress = "0x0000000000000000000000000000000000000400";
const precompileAbi = [
  "function bech32ToHex(string memory bech32Address) returns (address)"
];

// Create a contract instance
const contract = new ethers.Contract(precompileAddress, precompileAbi, provider);

// Input
const bech32Address = "Ontomir1mrdxhunfvjhe6lhdncp72dq46da2jcz9d9sh93"; // Placeholder

async function convertToHex() {
  try {
    const hexAddress = await contract.bech32ToHex.staticCall(bech32Address);
    console.log("Hex Address:", hexAddress);
  } catch (error) {
    console.error("Error converting to Hex:", error);
  }
}

convertToHex();
````

```
# Replace <RPC_URL> with your actual RPC endpoint.
# The address is a placeholder.
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "eth_call",
    "params": [
        {
            "to": "0x0000000000000000000000000000000000000400",
            "data": "0xe6df461e0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002a636f736d6f733171716c38616734636c757a367234647a32327064656e726d776c6975667173733863396a306500000000000000000000000000000000000000"
        },
        "latest"
    ],
    "id": 1
}' -H "Content-Type: application/json" <RPC_URL>
```

**Parameters**:

* `bech32Address` (string): The bech32 formatted address to convert

**Returns**: Ethereum address in hex format

### Full Interface & ABI <a href="#full-interface-andamp-abi" id="full-interface-andamp-abi"></a>

```
// SPDX-License-Identifier: LGPL-3.0-only
pragma solidity >=0.8.17;

/// @dev The Bech32I contract's address.
address constant Bech32_PRECOMPILE_ADDRESS = 0x0000000000000000000000000000000000000400;

/// @dev The Bech32I contract's instance.
Bech32I constant BECH32_CONTRACT = Bech32I(Bech32_PRECOMPILE_ADDRESS);

interface Bech32I {
    function hexToBech32(address addr,string memory prefix) external returns (string memory bech32Address);
    function bech32ToHex(string memory bech32Address) external returns (address addr);
}
```

```
{
  "_format": "hh-sol-artifact-1",
  "contractName": "Bech32I",
  "sourceName": "solidity/precompiles/bech32/Bech32I.sol",
  "abi": [
    {
      "inputs": [
        {
          "internalType": "string",
          "name": "bech32Address",
          "type": "string"
        }
      ],
      "name": "bech32ToHex",
      "outputs": [
        {
          "internalType": "address",
          "name": "addr",
          "type": "address"
        }
      ],
      "stateMutability": "nonpayable",
      "type": "function"
    },
    {
      "inputs": [
        {
          "internalType": "address",
          "name": "addr",
          "type": "address"
        },
        {
          "internalType": "string",
          "name": "prefix",
          "type": "string"
        }
      ],
      "name": "hexToBech32",
      "outputs": [
        {
          "internalType": "string",
          "name": "bech32Address",
          "type": "string"
        }
      ],
      "stateMutability": "nonpayable",
      "type": "function"
    }
  ],
  "bytecode": "0x",
  "deployedBytecode": "0x",
  "linkReferences": {},
  "deployedLinkReferences": {}
}
```

### Implementation Details <a href="#implementation-details" id="implementation-details"></a>

#### Address Validation <a href="#address-validation" id="address-validation"></a>

Both methods perform validation on the address format:

* **hexToBech32**: Validates that the hex address is exactly 20 bytes and the prefix is non-empty
* **bech32ToHex**: Validates that the bech32 address contains the separator character "1" and decodes to 20 bytes

#### Prefix Handling <a href="#prefix-handling" id="prefix-handling"></a>

For `hexToBech32`:

* The prefix parameter determines the human-readable part of the bech32 address
* Common prefixes include account addresses, validator addresses, and consensus addresses
* Empty or whitespace-only prefixes result in an error with suggested valid prefixes

For `bech32ToHex`:

* The prefix is automatically extracted from the bech32 address
* No prefix parameter is required as it's embedded in the address

#### State Mutability <a href="#state-mutability" id="state-mutability"></a>

While both methods are marked as `nonpayable` in the ABI, they function as read-only operations and do not modify blockchain state.
