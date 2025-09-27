# Multicall3

> Batch multiple contract calls in a single transaction for gas efficiency and atomicity

Multicall3 is a utility contract that enables batching multiple contract calls into a single transaction. This provides gas savings, atomic execution, and simplified interaction patterns for complex multi-step operations.

**Contract Adssdress**: `0xcA11bde05977b3631167028862bE2a173976CA11` **Deployment Status**: Default preinstall

### Key Features <a href="#key-features" id="key-features"></a>

* **Gas Efficiency**: Save on base transaction costs by batching calls
* **Atomic Execution**: All calls succeed or all revert together (optional)
* **State Consistency**: Read multiple values from the same block
* **Flexible Error Handling**: Choose between strict or permissive execution
* **Value Forwarding**: Send ETH/native tokens with calls

### Core Methods <a href="#core-methods" id="core-methods"></a>

#### `aggregate` <a href="#aggregate" id="aggregate"></a>

Execute multiple calls and revert if any fail.

```
function aggregate(Call[] calldata calls)
    returns (uint256 blockNumber, bytes[] memory returnData)
```

#### `tryAggregate` <a href="#tryaggregate" id="tryaggregate"></a>

Execute multiple calls, returning success status for each.

```
function tryAggregate(bool requireSuccess, Call[] calldata calls)
    returns (Result[] memory returnData)
```

#### `aggregate3` <a href="#aggregate3" id="aggregate3"></a>

Most flexible method with per-call configuration.

```
function aggregate3(Call3[] calldata calls)
    returns (Result[] memory returnData)
```

#### `aggregate3Value` <a href="#aggregate3value" id="aggregate3value"></a>

Like aggregate3 but allows sending value with each call.

```
function aggregate3Value(Call3Value[] calldata calls)
    payable returns (Result[] memory returnData)
```

### Usage Examples <a href="#usage-examples" id="usage-examples"></a>

#### Basic Batch Calls <a href="#basic-batch-calls" id="basic-batch-calls"></a>

\`\`\`javascript "Ethers.js Multiple Token Balance Query" expandable import { ethers } from "ethers";

const provider = new ethers.JsonRpcProvider("YOUR\_RPC\_URL"); const MULTICALL3 = "0xcA11bde05977b3631167028862bE2a173976CA11";

// Multicall3 ABI const multicallAbi = \[ "function aggregate3(tuple(address target, bool allowFailure, bytes callData)\[] calls) returns (tuple(bool success, bytes returnData)\[] returnData)" ];

const multicall = new ethers.Contract(MULTICALL3, multicallAbi, provider);

// Example: Read multiple token balances async function getMultipleBalances(tokenAddress, accounts) { const erc20Abi = \["function balanceOf(address) view returns (uint256)"]; const iface = new ethers.Interface(erc20Abi); // Prepare calls const calls = accounts.map(account => ({ target: tokenAddress, allowFailure: false, callData: iface.encodeFunctionData("balanceOf", \[account]) })); // Execute multicall const results = await multicall.aggregate3(calls); // Decode results const balances = results.map((result, i) => { if (result.success) { return { account: accounts\[i], balance: iface.decodeFunctionResult("balanceOf", result.returnData)\[0] }; } return { account: accounts\[i], balance: null }; }); return balances;

}

````

```solidity "Solidity Multi-Token Balance Reader" expandable
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IMulticall3 {
    struct Call3 {
        address target;
        bool allowFailure;
        bytes callData;
    }

    struct Result {
        bool success;
        bytes returnData;
    }

    function aggregate3(Call3[] calldata calls)
        external returns (Result[] memory returnData);
}

contract MultiTokenReader {
    IMulticall3 constant multicall = IMulticall3(0xcA11bde05977b3631167028862bE2a173976CA11);

    function getTokenBalances(
        address[] calldata tokens,
        address account
    ) external returns (uint256[] memory balances) {
        IMulticall3.Call3[] memory calls = new IMulticall3.Call3[](tokens.length);

        for (uint i = 0; i < tokens.length; i++) {
            calls[i] = IMulticall3.Call3({
                target: tokens[i],
                allowFailure: true,
                callData: abi.encodeWithSignature("balanceOf(address)", account)
            });
        }

        IMulticall3.Result[] memory results = multicall.aggregate3(calls);
        balances = new uint256[](results.length);

        for (uint i = 0; i < results.length; i++) {
            if (results[i].success) {
                balances[i] = abi.decode(results[i].returnData, (uint256));
            }
        }
    }
}
````

#### Complex DeFi Operations <a href="#complex-defi-operations" id="complex-defi-operations"></a>

Combine multiple DeFi operations atomically:

```
// Swap and stake in one transaction
async function swapAndStake(tokenIn, tokenOut, amountIn, poolAddress) {
  const calls = [
    // 1. Approve router
    {
      target: tokenIn,
      allowFailure: false,
      callData: iface.encodeFunctionData("approve", [router, amountIn])
    },
    // 2. Perform swap
    {
      target: router,
      allowFailure: false,
      callData: iface.encodeFunctionData("swap", [tokenIn, tokenOut, amountIn])
    },
    // 3. Approve staking
    {
      target: tokenOut,
      allowFailure: false,
      callData: iface.encodeFunctionData("approve", [poolAddress, amountOut])
    },
    // 4. Stake tokens
    {
      target: poolAddress,
      allowFailure: false,
      callData: iface.encodeFunctionData("stake", [amountOut])
    }
  ];

  return await multicall.aggregate3(calls);
}
```

#### Reading Protocol State <a href="#reading-protocol-state" id="reading-protocol-state"></a>

Get consistent protocol state in one call:

```
async function getProtocolState(protocol) {
  const calls = [
    { target: protocol, callData: "0x18160ddd", allowFailure: false }, // totalSupply()
    { target: protocol, callData: "0x313ce567", allowFailure: false }, // decimals()
    { target: protocol, callData: "0x06fdde03", allowFailure: false }, // name()
    { target: protocol, callData: "0x95d89b41", allowFailure: false }, // symbol()
  ];

  const results = await multicall.aggregate3(calls);
  // All values from the same block
  return decodeResults(results);
}
```

### Advanced Features <a href="#advanced-features" id="advanced-features"></a>

#### Value Forwarding <a href="#value-forwarding" id="value-forwarding"></a>

Send native tokens with calls using `aggregate3Value`:

```
const calls = [
  {
    target: weth,
    allowFailure: false,
    value: ethers.parseEther("1.0"),
    callData: iface.encodeFunctionData("deposit")
  },
  {
    target: contract2,
    allowFailure: false,
    value: ethers.parseEther("0.5"),
    callData: iface.encodeFunctionData("buyTokens")
  }
];

await multicall.aggregate3Value(calls, {
  value: ethers.parseEther("1.5") // Total ETH to send
});
```

#### Error Handling Strategies <a href="#error-handling-strategies" id="error-handling-strategies"></a>

```
// Strict mode - revert if any call fails
const strictCalls = calls.map(call => ({
  ...call,
  allowFailure: false
}));

// Permissive mode - continue even if some fail
const permissiveCalls = calls.map(call => ({
  ...call,
  allowFailure: true
}));

// Mixed mode - critical calls must succeed
const mixedCalls = [
  { ...criticalCall, allowFailure: false },
  { ...optionalCall, allowFailure: true }
];
```

### Common Use Cases <a href="#common-use-cases" id="common-use-cases"></a>

Query multiple token balances for multiple users efficiently. Fetch prices from multiple DEXs in a single call for best execution. Cast votes on multiple proposals in one transaction. Claim rewards, compound, and rebalance positions atomically. Mint, transfer, or approve multiple NFTs efficiently.

### Gas Optimization <a href="#gas-optimization" id="gas-optimization"></a>

Multicall3 saves gas through:

1. **Base Fee Savings**: Pay the 21,000 gas base fee only once
2. **Storage Optimization**: Warm storage slots across calls
3. **Reduced State Changes**: Fewer transaction state transitions

Typical savings: 20-40% for batching 5+ calls

### Best Practices <a href="#best-practices" id="best-practices"></a>

* **Batch Size**: Optimal batch size is 10-50 calls depending on complexity
* **Gas Limits**: Set appropriate gas limits for complex batches
* **Error Handling**: Use `allowFailure` wisely based on criticality
* **Return Data**: Decode return data carefully, checking success flags
* **Reentrancy**: Be aware of potential reentrancy in batched calls

### Comparison with Alternatives <a href="#comparison-with-alternatives" id="comparison-with-alternatives"></a>

| Feature           | Multicall3       | Multicall2 | Manual Batching |
| ----------------- | ---------------- | ---------- | --------------- |
| Gas Efficiency    | Excellent        | Good       | Poor            |
| Error Handling    | Flexible         | Basic      | N/A             |
| Value Support     | Yes              | No         | Yes             |
| Deployment        | Standard address | Varies     | N/A             |
| Block Consistency | Yes              | Yes        | No              |

### Integration Libraries <a href="#integration-libraries" id="integration-libraries"></a>

Popular libraries with Multicall3 support:

* **ethers-rs**: Rust implementation
* **web3.py**: Python Web3 library
* **viem**: TypeScript alternative to ethers
* **wagmi**: React hooks for Ethereum

### Troubleshooting <a href="#troubleshooting" id="troubleshooting"></a>

| Issue                     | Solution                                                 |
| ------------------------- | -------------------------------------------------------- |
| "Multicall3: call failed" | Check individual call success flags                      |
| Gas estimation failure    | Increase gas limit or reduce batch size                  |
| Unexpected revert         | One of the calls with `allowFailure: false` failed       |
| Value mismatch            | Ensure total value sent matches sum of individual values |

### Further Reading <a href="#further-reading" id="further-reading"></a>

* [Multicall3 GitHub Repository](https://github.com/mds1/multicall)
* [Multicall3 Deployments](https://www.multicall3.com/)
* [Original Multicall by MakerDAO](https://github.com/makerdao/multicall)
