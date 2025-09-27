# Accounts

> Crypto Wallets (or Accounts) can be created and represented in unique ways on different blockchains. For developers who interface with account types on a Ontomir EVM chain, e.g. during wallet integration on their dApp frontend, it is therefore important to understand that accounts in the Ontomir EVM are implemented to be compatible with Ethereum type addresses.

### Prerequisite Readings <a href="#prerequisite-readings" id="prerequisite-readings"></a>

Learn about account structure in Ontomir SDK Understand Ethereum account model

### Creating Accounts <a href="#creating-accounts" id="creating-accounts"></a>

To create one account you can either create a private key, a keystore file (a private key protected by a password), or a mnemonic phrase (a string of words that can access multiple private keys).

Aside from having different security features, the biggest difference between each of these is that a private key or keystore file only creates one account. Creating a mnemonic phrase gives you control of many accounts, all accessible with that same phrase.



### Representing Accounts <a href="#representing-accounts" id="representing-accounts"></a>

The terms "account" and "address" are often used interchangeably to describe crypto wallets. In the Ontomir SDK, an account designates a pair of public key (PubKey) and private key (PrivKey). The derivation path defines what the private key, public key, and address would be.

The PubKey can be derived to generate various addresses in different formats, which are used to identify users (among other parties) in the application. A common address form for Ontomir chains is the bech32 format (e.g. `Ontomir1...`). Addresses are also associated with messages to identify the sender of the message.

The PrivKey is used to generate digital signatures to prove that an address associated with the PrivKey approved of a given message. The proof is performed by applying a cryptographic scheme to the PrivKey, known as Elliptic Curve Digital Signature Algorithm (ECDSA), to generate a PubKey that is compared with the address in the message.

### EVM Accounts <a href="#evm-accounts" id="evm-accounts"></a>

Ontomir EVM defines its own custom `Account` type to implement a HD wallet that is compatible with Ethereum type addresses. It uses Ethereum's ECDSA secp256k1 curve for keys (`eth_secp265k1`) and satisfies the [EIP84](https://github.com/ethereum/EIPs/issues/84) for full [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) paths. This cryptographic curve is not to be confused with [Bitcoin's ECDSA secp256k1](https://en.bitcoin.it/wiki/Secp256k1) curve.

The root HD path for EVM-based accounts is `m/44'/60'/0'/0`. It is recommended to use the Coin type `60` to support Ethereum type accounts, unlike many other Ontomir chains that use Coin type `118` ([list of coin types](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)

The custom Ontomir EVM [EthAccount](https://github.com/Ontomir/evm/blob/v0.4.1/types/account.go#L28-L33) satisfies the `AccountI` interface from the Ontomir SDK auth module and includes additional fields that are required for Ethereum type addresses:

```
// EthAccountI represents the interface of an EVM compatible account
type EthAccountI interface {
    authtypes.AccountI

    // EthAddress returns the ethereum Address representation of the AccAddress
    EthAddress() common.Address

    // CodeHash is the keccak256 hash of the contract code (if any)
    GetCodeHash() common.Hash

    // SetCodeHash sets the code hash to the account fields
    SetCodeHash(code common.Hash) error

    // Type returns the type of Ethereum Account (EOA or Contract)
    Type() int8
}
```

**EIP-7702 Support**: Ontomir EVM supports EIP-7702 code delegation, allowing EOAs to temporarily delegate code execution to smart contracts through the `SetCodeHash` functionality.

For more information on Ethereum accounts head over to the x/vm module.

#### Addresses and Public Keys <a href="#addresses-and-public-keys" id="addresses-and-public-keys"></a>

[BIP-0173](https://github.com/satoshilabs/slips/blob/master/slip-0173.md) defines a new format for segregated witness output addresses that contains a human-readable part that identifies the Bech32 usage.

There are 3 main types of HRP for the `Addresses`/`PubKeys` available by default on the Ontomir EVM:

\*\*Purpose\*\*: Identify users (e.g., transaction senders)

```
* **Curve**: `eth_secp256k1`
* **Address Prefix**: `Ontomir`
* **Pubkey Prefix**: `Ontomirpub`
* **Address Length**: 20 bytes
* **Pubkey Length**: 33 bytes (compressed)
```

\*\*Purpose\*\*: Identify validator operators

```
* **Curve**: `eth_secp256k1`
* **Address Prefix**: `Ontomirvaloper`
* **Pubkey Prefix**: `Ontomirvaloperpub`
* **Address Length**: 20 bytes
* **Pubkey Length**: 33 bytes (compressed)
```

\*\*Purpose\*\*: Identify nodes participating in consensus

```
* **Curve**: `ed25519`
* **Address Prefix**: `Ontomirvalcons`
* **Pubkey Prefix**: `Ontomirvalconspub`
* **Address Length**: 20 bytes
* **Pubkey Length**: 32 bytes
```

#### Address formats for clients <a href="#address-formats-for-clients" id="address-formats-for-clients"></a>

`EthAccount` can be represented in both [Bech32](https://en.bitcoin.it/wiki/Bech32) (e.g. `Ontomir1...`) and hex (`0x...`) formats for Ethereum's Web3 tooling compatibility.

The Bech32 format is the default for Ontomir-SDK queries and transactions, while the hex format is used for Ethereum compatibility. \`\`\`text Bech32 Address Ontomir1z3t55m0l9h0eupuz3dp5t5cypyv674jj7mz2jw \`\`\`

```
0x91defC7fE5603DFA8CC9B655cF5772459BF10c6f
```

```
{
  "@type": "/ethermint.crypto.v1.ethsecp256k1.PubKey",
  "key": "AsV5oddeB+hkByIJo/4lZiVUgXTzNfBPKC73cZ4K1YD2"
}
```

#### Address conversion <a href="#address-conversion" id="address-conversion"></a>

The `evmd debug addr <address>` can be used to convert an address between hex and bech32 formats:

\`\`\`bash Bech32 to Hex $ evmd debug addr Ontomir1z3t55m0l9h0eupuz3dp5t5cypyv674jj7mz2jw

Address: \[20 87 74 109 255 45 223 158 7 130 139 67 69 211 4 9 25 175 86 82] Address (hex): 14574A6DFF2DDF9E07828B4345D3040919AF5652 Bech32 Acc: Ontomir1z3t55m0l9h0eupuz3dp5t5cypyv674jj7mz2jw Bech32 Val: Ontomirvaloper1z3t55m0l9h0eupuz3dp5t5cypyv674jjn4d6nn

````

```bash Hex to Bech32
$ evmd debug addr 14574A6DFF2DDF9E07828B4345D3040919AF5652

Address: [20 87 74 109 255 45 223 158 7 130 139 67 69 211 4 9 25 175 86 82]
Address (hex): 14574A6DFF2DDF9E07828B4345D3040919AF5652
Bech32 Acc: Ontomir1z3t55m0l9h0eupuz3dp5t5cypyv674jj7mz2jw
Bech32 Val: Ontomirvaloper1z3t55m0l9h0eupuz3dp5t5cypyv674jjn4d6nn
````

#### Key output <a href="#key-output" id="key-output"></a>

The Ontomir SDK Keyring output (i.e \`evmd keys\`) only supports addresses and public keys in Bech32 format.

We can use the `keys show` command with the flag `--bech <type>` to obtain different address formats:

\`\`\`bash $ evmd keys show dev0 --bech acc

````
- name: dev0
  type: local
  address: Ontomir1z3t55m0l9h0eupuz3dp5t5cypyv674jj7mz2jw
  pubkey: '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"AsV5oddeB+hkByIJo/4lZiVUgXTzNfBPKC73cZ4K1YD2"}'
  mnemonic: ""
```
````

\`\`\`bash $ evmd keys show dev0 --bech val

````
- name: dev0
  type: local
  address: Ontomirvaloper1z3t55m0l9h0eupuz3dp5t5cypyv674jjn4d6nn
  pubkey: '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"AsV5oddeB+hkByIJo/4lZiVUgXTzNfBPKC73cZ4K1YD2"}'
  mnemonic: ""
```
````

\`\`\`bash $ evmd keys show dev0 --bech cons

````
- name: dev0
  type: local
  address: Ontomirvalcons1rllqa5d97n6zyjhy6cnscc7zu30zjn3f7wyj2n
  pubkey: '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"A/fVLgIqiLykFQxum96JkSOoTemrXD0tFaFQ1B0cpB2c"}'
  mnemonic: ""
```
````

### Querying an Account <a href="#querying-an-account" id="querying-an-account"></a>

You can query an account address using CLI, gRPC, REST, or JSON-RPC:

\`\`\`bash "Query Account via CLI" expandable # NOTE: the --output (-o) flag defines the output format evmd q auth account $(evmd keys show dev0 -a) -o text

````
'@type': /ethermint.types.v1.EthAccount
base_account:
  account_number: "0"
  address: Ontomir1z3t55m0l9h0eupuz3dp5t5cypyv674jj7mz2jw
  pub_key:
    '@type': /ethermint.crypto.v1.ethsecp256k1.PubKey
    key: AsV5oddeB+hkByIJo/4lZiVUgXTzNfBPKC73cZ4K1YD2
  sequence: "1"
code_hash: 0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470
```
````

\`\`\`bash # GET /Ontomir/auth/v1beta1/accounts/{address} curl -X GET "http://localhost:10337/Ontomir/auth/v1beta1/accounts/Ontomir14au322k9munkmx5wrchz9q30juf5wjgz2cfqku" \ -H "accept: application/json" \`\`\` \`\`\`bash "JSON-RPC Account Queries" expandable # List accounts using eth\_accounts curl -X POST --data '{ "jsonrpc":"2.0", "method":"eth\_accounts", "params":\[], "id":1 }' -H "Content-Type: application/json" http://localhost:8545

````
# Or using personal_listAccounts
curl -X POST --data '{
  "jsonrpc":"2.0",
  "method":"personal_listAccounts",
  "params":[],
  "id":1
}' -H "Content-Type: application/json" http://localhost:8545
```
````

For JSON-RPC methods, see \[\`eth\_accounts\`]\(/docs/evm/next/api-reference/ethereum-json-rpc/methods#eth-accounts) and \[\`personal\_listAccounts\`]\(/docs/evm/next/api-reference/ethereum-json-rpc/methods#personal-listAccounts) documentation.
