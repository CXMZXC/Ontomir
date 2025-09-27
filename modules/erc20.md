# ERC20

> Token pair management and conversion between Ontomir coins and ERC20 tokens

The `x/erc20` module from Ontomir evm enables bidirectional conversion between Ontomir SDK coins and ERC20 tokens within the EVM runtime.

For conceptual understanding of Single Token Representation v2, see \[Single Token Representation]\(/docs/evm/next/documentation/concepts/single-token-representation).

### Parameters <a href="#parameters" id="parameters"></a>

The module parameters control token conversion and registration :

| Parameter                     | Type | Default | Description                           |
| ----------------------------- | ---- | ------- | ------------------------------------- |
| `enable_erc20`                | bool | `true`  | Enable token conversions globally     |
| `permissionless_registration` | bool | `false` | Allow anyone to register ERC20 tokens |

#### Parameter Details <a href="#parameter-details" id="parameter-details"></a>

Master switch for token conversions:

* `true`: All conversions enabled (default)
* `false`: Disables all conversions, registration still works
* Use for emergency pause or maintenance

Controls who can register new ERC20 tokens:

* `false`: Only governance can register (default, recommended)
* `true`: Anyone can register via MsgRegisterERC20
* Permissionless risks: spam tokens, malicious contracts

### State <a href="#state" id="state"></a>

The module maintains token pair mappings and allowances :

| Object               | Key              | Value       | Description                 |
| -------------------- | ---------------- | ----------- | --------------------------- |
| `TokenPair`          | `0x01 + ID`      | `TokenPair` | Token pair configuration    |
| `TokenPairByERC20`   | `0x02 + address` | `ID`        | Lookup by ERC20 address     |
| `TokenPairByDenom`   | `0x03 + denom`   | `ID`        | Lookup by denomination      |
| `Allowance`          | `0x04 + hash`    | `Allowance` | ERC20 allowances            |
| `NativePrecompiles`  | `0x05 + address` | `bool`      | Native precompile registry  |
| `DynamicPrecompiles` | `0x06 + address` | `bool`      | Dynamic precompile registry |

#### Token Pair Structure <a href="#token-pair-structure" id="token-pair-structure"></a>

```
type TokenPair struct {
    Erc20Address  string  // Hex address of ERC20 contract
    Denom         string  // Ontomir coin denomination
    Enabled       bool    // Conversion enable status
    ContractOwner Owner   // OWNER_MODULE or OWNER_EXTERNAL
}
```

### Token Pair Registration <a href="#token-pair-registration" id="token-pair-registration"></a>

#### Registration Methods <a href="#registration-methods" id="registration-methods"></a>

1. **Automatic Registration (IBC Tokens)**
   * IBC tokens (denoms starting with "ibc/") are automatically registered on first receipt
   * No governance proposal or user action required
   * Creates ERC20 precompile at deterministic address
2. **Permissionless Registration (ERC20 Contracts)**
   * When `permissionless_registration` parameter is `true`
   * Any user can register existing ERC20 contracts via `MsgRegisterERC20`
   * Useful for integrating existing ERC20 tokens
3. **Governance Registration**
   * Always available regardless of parameter settings
   * Can register any ERC20 contract or create new token pairs
   * Required when `permissionless_registration` is `false`

### Messages <a href="#messages" id="messages"></a>

#### MsgRegisterERC20 <a href="#msgregistererc20" id="msgregistererc20"></a>

Register existing ERC20 contracts for conversion :

```
message MsgRegisterERC20 {
    string signer = 1;
    repeated string erc20addresses = 2;
}
```

**Requirements:**

* `permissionless_registration` enabled OR sender is governance authority
* Valid ERC20 contract at address
* Contract not already registered
* Contract implements standard ERC20 interface

#### MsgConvertCoin <a href="#msgconvertcoin" id="msgconvertcoin"></a>

Convert Ontomir coins to ERC20 tokens:

```
message MsgConvertCoin {
    Coin coin = 1;      // Amount and denom to convert
    string receiver = 2; // Hex address to receive ERC20
    string sender = 3;   // Bech32 address of sender
}
```

**Validation:**

* Token pair exists and enabled
* Sender has sufficient balance
* Valid receiver address

#### MsgConvertERC20 <a href="#msgconverterc20" id="msgconverterc20"></a>

Convert ERC20 tokens to Ontomir coins:

```
message MsgConvertERC20 {
    string contract_address = 1;  // ERC20 contract
    string amount = 2;            // Amount to convert
    string receiver = 3;          // Bech32 address
    string sender = 4;            // Hex address of sender
}
```

**Validation:**

* Token pair exists and enabled
* Sender has sufficient ERC20 balance
* Valid receiver address

#### MsgToggleConversion <a href="#msgtoggleconversion" id="msgtoggleconversion"></a>

Enable/disable conversions for a token pair (governance only):

```
message MsgToggleConversion {
    string authority = 1;  // Must be governance account
    string token = 2;      // Address or denom
}
```

#### MsgUpdateParams <a href="#msgupdateparams" id="msgupdateparams"></a>

Update module parameters (governance only):

```
message MsgUpdateParams {
    string authority = 1;  // Must be governance account
    Params params = 2;     // New parameters
}
```

### Conversion Flows <a href="#conversion-flows" id="conversion-flows"></a>

#### Native Coin → ERC20 <a href="#native-coin-erc20" id="native-coin-erc20"></a>

UserModuleBankEVMMsgConvertCoinEscrow coinsMint ERC20ERC20 tokensUserModuleBankEVM

**Steps:**

1. Validate token pair enabled
2. Transfer coins to module account
3. Mint equivalent ERC20 to receiver
4. Emit conversion event

#### ERC20 → Native Coin <a href="#erc20-native-coin" id="erc20-native-coin"></a>

UserModuleEVMBankMsgConvertERC20Burn/Escrow tokensRelease coinsNative coinsUserModuleEVMBank

**Steps:**

1. Validate token pair enabled
2. For module-owned: burn ERC20
3. For external: transfer to module
4. Release native coins from escrow
5. Emit conversion event

### Precompile System <a href="#precompile-system" id="precompile-system"></a>

#### Native Precompiles <a href="#native-precompiles" id="native-precompiles"></a>

Automatically created for Ontomir coins at deterministic addresses:

```
// Address derivation
address = keccak256("erc20|<denom>")[:20]
```

**Interface:**

```
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
}
```

#### Dynamic Precompiles (WERC20) <a href="#dynamic-precompiles-werc20" id="dynamic-precompiles-werc20"></a>

Optional wrapped interface for registered tokens:

```
interface IWERC20 is IERC20 {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}
```

### IBC Integration <a href="#ibc-integration" id="ibc-integration"></a>

#### IBC Middleware v1 <a href="#ibc-middleware-v1" id="ibc-middleware-v1"></a>

Standard IBC transfer integration :

\`\`\`go // Automatic registration and conversion on receive OnRecvPacket(packet) { // Auto-register new IBC tokens (with "ibc/" prefix) if !tokenPairExists && hasPrefix(denom, "ibc/") { RegisterERC20Extension(denom) }

```
  // Auto-convert to ERC20 for Ethereum addresses
  if isEthereumAddress(receiver) {
      convertToERC20(voucher, receiver)
  }
```

}

// Automatic conversion on acknowledgment OnAcknowledgementPacket(packet, ack) { if wasConverted { convertBackToOntomir(refund) } }

````
</Expandable>

**Automatic Registration:**

* IBC tokens (denoms starting with "ibc/") are automatically registered on first receipt
* Factory tokens (denoms starting with "factory/") are skipped
* Native chain tokens require explicit registration

### IBC Middleware v2

Enhanced IBC v2 support ([source](https://github.com/Ontomir/evm/blob/v0.4.1/x/erc20/v2/ibc_middleware.go)):

* New packet format support
* Improved error handling
* Multi-hop awareness
* Backward compatibility

## Events

### Registration Events

| Event                     | Attributes                     | When Emitted          |
| ------------------------- | ------------------------------ | --------------------- |
| `register_erc20`          | `erc20_address`, `Ontomir_coin` | Token pair registered |
| `toggle_token_conversion` | `erc20_address`, `Ontomir_coin` | Conversion toggled    |

### Conversion Events

| Event           | Attributes                                                   | When Emitted |
| --------------- | ------------------------------------------------------------ | ------------ |
| `convert_coin`  | `sender`, `receiver`, `amount`, `Ontomir_coin`, `erc20_token` | Coin → ERC20 |
| `convert_erc20` | `sender`, `receiver`, `amount`, `Ontomir_coin`, `erc20_token` | ERC20 → Coin |

### IBC Events

| Event                     | Attributes                              | When Emitted               |
| ------------------------- | --------------------------------------- | -------------------------- |
| `ibc_transfer_conversion` | `sender`, `receiver`, `denom`, `amount` | Auto-conversion during IBC |

## Queries

### gRPC

```protobuf
service Query {
  // Get module parameters
  rpc Params(QueryParamsRequest) returns (QueryParamsResponse);

  // Get all token pairs
  rpc TokenPairs(QueryTokenPairsRequest) returns (QueryTokenPairsResponse);

  // Get specific token pair
  rpc TokenPair(QueryTokenPairRequest) returns (QueryTokenPairResponse);
}
````

#### CLI <a href="#cli" id="cli"></a>

\`\`\`bash Query-Params # Query module parameters evmd query erc20 params \`\`\`

```
# Query all registered token pairs
evmd query erc20 token-pairs

# Query specific token pair
evmd query erc20 token-pair test
evmd query erc20 token-pair 0x80b5a32e4f032b2a058b4f29ec95eefeeb87adcd
```

```
# Convert Ontomir coin to ERC20
evmd tx erc20 convert-coin 100test 0xReceiverAddress --from mykey

# Convert ERC20 to Ontomir coin
evmd tx erc20 convert-erc20 0xTokenAddress 100000000000000000000 Ontomir1receiver --from mykey
```

```
# Register ERC20 token (if permissionless)
evmd tx erc20 register-erc20 0xContractAddress --from mykey

# Via governance (if permissioned)
evmd tx gov submit-proposal register-erc20-proposal.json --from mykey
```

### Integration Examples <a href="#integration-examples" id="integration-examples"></a>

#### DeFi Protocol Integration <a href="#defi-protocol-integration" id="defi-protocol-integration"></a>

```
// Using precompiled ERC20 interface for IBC tokens
const ibcToken = new ethers.Contract(
    "0x80b5a32e4f032b2a058b4f29ec95eefeeb87adcd", // Precompile address
    ERC20_ABI,
    signer
);

// Standard ERC20 operations work seamlessly
await ibcToken.approve(dexRouter, amount);
await dexRouter.swapExactTokensForTokens(...);
```

#### Automatic IBC Conversion <a href="#automatic-ibc-conversion" id="automatic-ibc-conversion"></a>

```
// IBC transfer automatically converts to ERC20
const packet = {
    sender: "Ontomir1...",
    receiver: "0x...",  // EVM address triggers conversion
    token: { denom: "test", amount: "1000000" }
};
// Token arrives as ERC20 in receiver's EVM account
```

#### Manual Conversion Flow <a href="#manual-conversion-flow" id="manual-conversion-flow"></a>

```
// Convert native to ERC20
await client.signAndBroadcast(address, [
    {
        typeUrl: "/Ontomir.evm.erc20.v1.MsgConvertCoin",
        value: {
            coin: { denom: "test", amount: "1000000" },
            receiver: "0xEthereumAddress",
            sender: "Ontomir1..."
        }
    }
]);
```

### Best Practices <a href="#best-practices" id="best-practices"></a>

#### Chain Integration <a href="#chain-integration" id="chain-integration"></a>

1. **Token Registration Review**
   * Audit contracts before registration
   * Verify standard compliance
   * Check for malicious behavior
2. **Precompile Configuration**
   * Enable precompiles for frequently used tokens
   * Monitor gas consumption
   * Set appropriate gas costs
3. **IBC Setup**
   * Configure middleware stack correctly
   * Test auto-conversion flows
   * Monitor conversion events

#### Security Considerations <a href="#security-considerations" id="security-considerations"></a>

1.  **Contract Validation**

    ```
    // Verify standard interface
    IERC20(token).totalSupply();
    IERC20(token).balanceOf(address(this));
    ```
2. **Event Monitoring**
   * Track conversion events
   * Monitor for unusual patterns
   * Alert on large conversions
3. **Emergency Response**
   * Disable conversions via governance
   * Toggle specific token pairs
   * Have incident response plan

### Troubleshooting <a href="#troubleshooting" id="troubleshooting"></a>

#### Common Issues <a href="#common-issues" id="common-issues"></a>

| Issue                  | Cause                      | Solution                            |
| ---------------------- | -------------------------- | ----------------------------------- |
| "token pair not found" | Token not registered       | Register via governance             |
| "token pair disabled"  | Conversions toggled off    | Enable via governance               |
| "insufficient balance" | Low balance for conversion | Check balance in correct format     |
| "invalid recipient"    | Wrong address format       | Use hex for EVM, bech32 for Ontomir |
| "module disabled"      | enable\_erc20 = false      | Enable via governance               |

#### Debug Commands <a href="#debug-commands" id="debug-commands"></a>

```
# Check if token is registered
evmd query erc20 token-pair test

# Check module parameters
evmd query erc20 params

# Check account balance (both formats)
evmd query bank balances Ontomir1...
evmd query vm balance 0x...
```
