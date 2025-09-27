# Permit2

> Universal token approval and transfer management system with signature-based permissions

Permit2 is a universal token approval contract that enables signature-based approvals and transfers. Developed by Uniswap Labs, it improves upon EIP-2612 by providing a single contract for managing token permissions across all ERC20 tokens, even those that don't natively support permits.

**Contract Address**: `0x000000000022D473030F116dDEE9F6B43aC78BA3` **Deployment Status**: Default preinstall

### Key Benefits <a href="#key-benefits" id="key-benefits"></a>

* **Universal Compatibility**: Works with any ERC20 token
* **Gas Savings**: One-time approval to Permit2, then signature-based transfers
* **Enhanced Security**: Granular permissions with expiration times
* **Better UX**: Gasless approvals via signatures
* **Batching Support**: Multiple token operations in one transaction

### Core Concepts <a href="#core-concepts" id="core-concepts"></a>

#### Two-Step Process <a href="#two-step-process" id="two-step-process"></a>

1. **One-time Approval**: User approves Permit2 to spend their tokens
2. **Signature Permits**: User signs messages for specific transfers

#### Permission Types <a href="#permission-types" id="permission-types"></a>

* **AllowanceTransfer**: Traditional allowance model with signatures
* **SignatureTransfer**: Direct transfers using only signatures

### Core Methods <a href="#core-methods" id="core-methods"></a>

#### Allowance Transfer <a href="#allowance-transfer" id="allowance-transfer"></a>

```
// Grant permission via signature
function permit(
    address owner,
    PermitSingle memory permitSingle,
    bytes calldata signature
) external

// Transfer with existing permission
function transferFrom(
    address from,
    address to,
    uint160 amount,
    address token
) external
```

#### Signature Transfer <a href="#signature-transfer" id="signature-transfer"></a>

```
// One-time transfer via signature
function permitTransferFrom(
    PermitTransferFrom memory permit,
    SignatureTransferDetails calldata transferDetails,
    address owner,
    bytes calldata signature
) external
```

### Usage Examples <a href="#usage-examples" id="usage-examples"></a>

#### Basic Permit2 Integration <a href="#basic-permit2-integration" id="basic-permit2-integration"></a>

\`\`\`javascript "Ethers.js Complete Permit2 Integration" expandable import { ethers } from "ethers"; import { AllowanceProvider, AllowanceTransfer } from "@uniswap/permit2-sdk";

const PERMIT2\_ADDRESS = "0x000000000022D473030F116dDEE9F6B43aC78BA3";

// Step 1: User approves Permit2 for token (one-time) async function approvePermit2(tokenContract, signer) { const tx = await tokenContract.approve( PERMIT2\_ADDRESS, ethers.MaxUint256 ); await tx.wait(); console.log("Permit2 approved for token"); }

// Step 2: Create and sign permit async function createPermit(token, spender, amount, deadline, signer) { const permit = { details: { token: token, amount: amount, expiration: deadline, nonce: 0, // Get current nonce from contract }, spender: spender, sigDeadline: deadline, }; // Create permit data const { domain, types, values } = AllowanceTransfer.getPermitData( permit, PERMIT2\_ADDRESS, await signer.provider.getNetwork().then(n => n.chainId) ); // Sign permit const signature = await signer.\_signTypedData(domain, types, values); return { permit, signature };

}

// Step 3: Execute transfer with permit async function transferWithPermit(permit, signature, transferDetails) { const permit2 = new ethers.Contract( PERMIT2\_ADDRESS, \["function transferFrom(address,address,uint160,address)"], signer ); // First, submit the permit await permit2.permit( signer.address, permit, signature ); // Then transfer await permit2.transferFrom( transferDetails.from, transferDetails.to, transferDetails.amount, transferDetails.token );

}

````

```solidity "Solidity Permit2 Integration Contract" expandable
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPermit2 {
    struct PermitDetails {
        address token;
        uint160 amount;
        uint48 expiration;
        uint48 nonce;
    }

    struct PermitSingle {
        PermitDetails details;
        address spender;
        uint256 sigDeadline;
    }

    struct SignatureTransferDetails {
        address to;
        uint256 requestedAmount;
    }

    function permit(
        address owner,
        PermitSingle memory permitSingle,
        bytes calldata signature
    ) external;

    function transferFrom(
        address from,
        address to,
        uint160 amount,
        address token
    ) external;
}

contract Permit2Integration {
    IPermit2 constant permit2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);

    function executeTransferWithPermit(
        address token,
        address from,
        address to,
        uint160 amount,
        IPermit2.PermitSingle calldata permit,
        bytes calldata signature
    ) external {
        // Submit permit
        permit2.permit(from, permit, signature);

        // Execute transfer
        permit2.transferFrom(from, to, amount, token);
    }
}
````

#### Batch Operations <a href="#batch-operations" id="batch-operations"></a>

Handle multiple tokens in one transaction:

```
async function batchTransferWithPermit2(transfers, owner, signer) {
  const permits = [];
  const signatures = [];

  // Prepare batch permits
  for (const transfer of transfers) {
    const permit = {
      details: {
        token: transfer.token,
        amount: transfer.amount,
        expiration: transfer.expiration,
        nonce: await getNonce(transfer.token, owner),
      },
      spender: transfer.spender,
      sigDeadline: transfer.deadline,
    };

    const signature = await signPermit(permit, signer);
    permits.push(permit);
    signatures.push(signature);
  }

  // Execute batch
  const permit2 = new ethers.Contract(PERMIT2_ADDRESS, abi, signer);
  await permit2.permitBatch(owner, permits, signatures);
}
```

#### Gasless Approvals <a href="#gasless-approvals" id="gasless-approvals"></a>

Enable gasless token approvals using meta-transactions:

```
// User signs permit off-chain
async function createGaslessPermit(token, spender, amount, signer) {
  const deadline = Math.floor(Date.now() / 1000) + 3600; // 1 hour

  const permit = {
    details: {
      token: token,
      amount: amount,
      expiration: deadline,
      nonce: 0,
    },
    spender: spender,
    sigDeadline: deadline,
  };

  const signature = await signPermit(permit, signer);

  // Return data for relayer
  return {
    permit,
    signature,
    owner: signer.address,
  };
}

// Relayer submits transaction
async function relayPermit(permitData, relayerSigner) {
  const permit2 = new ethers.Contract(PERMIT2_ADDRESS, abi, relayerSigner);

  const tx = await permit2.permit(
    permitData.owner,
    permitData.permit,
    permitData.signature
  );

  return tx.wait();
}
```

### Advanced Features <a href="#advanced-features" id="advanced-features"></a>

#### Witness Data <a href="#witness-data" id="witness-data"></a>

Include additional data in permits for complex protocols:

```
struct PermitWitnessTransferFrom {
    TokenPermissions permitted;
    address spender;
    uint256 nonce;
    uint256 deadline;
    bytes32 witness; // Custom data hash
}
```

#### Nonce Management <a href="#nonce-management" id="nonce-management"></a>

Permit2 uses unordered nonces for flexibility:

```
// Invalidate specific nonces
await permit2.invalidateNonces(token, spender, newNonce);

// Invalidate nonce range
await permit2.invalidateUnorderedNonces(wordPos, mask);
```

#### Expiration Handling <a href="#expiration-handling" id="expiration-handling"></a>

Set appropriate expiration times:

```
const expirations = {
  shortTerm: Math.floor(Date.now() / 1000) + 300,      // 5 minutes
  standard: Math.floor(Date.now() / 1000) + 3600,      // 1 hour
  longTerm: Math.floor(Date.now() / 1000) + 86400,     // 1 day
  maximum: 2n ** 48n - 1n,                             // Max allowed
};
```

### Security Considerations <a href="#security-considerations" id="security-considerations"></a>

#### Best Practices <a href="#best-practices" id="best-practices"></a>

1. **Validate Signatures**: Always verify signature validity
2. **Check Expiration**: Ensure permits haven't expired
3. **Nonce Tracking**: Prevent replay attacks
4. **Amount Limits**: Set reasonable amount limits
5. **Deadline Checks**: Validate signature deadlines

#### Common Attack Vectors <a href="#common-attack-vectors" id="common-attack-vectors"></a>

* **Signature Replay**: Mitigated by nonces
* **Front-running**: Use commit-reveal or deadlines
* **Phishing**: Educate users about signing
* **Unlimited Approvals**: Set specific amounts

### Integration Patterns <a href="#integration-patterns" id="integration-patterns"></a>

#### DEX Integration <a href="#dex-integration" id="dex-integration"></a>

```
contract DEXWithPermit2 {
    function swapWithPermit(
        SwapParams calldata params,
        PermitSingle calldata permit,
        bytes calldata signature
    ) external {
        // Get tokens via permit
        permit2.permit(msg.sender, permit, signature);
        permit2.transferFrom(
            msg.sender,
            address(this),
            permit.details.amount,
            permit.details.token
        );

        // Execute swap
        _performSwap(params);
    }
}
```

#### Payment Processor <a href="#payment-processor" id="payment-processor"></a>

```
class PaymentProcessor {
  async processPayment(order, permit, signature) {
    // Verify order details
    if (!this.verifyOrder(order)) {
      throw new Error("Invalid order");
    }

    // Process payment via Permit2
    await this.permit2.permitTransferFrom(
      permit,
      {
        to: this.treasury,
        requestedAmount: order.amount,
      },
      order.buyer,
      signature
    );

    // Fulfill order
    await this.fulfillOrder(order);
  }
}
```

### Comparison with Alternatives <a href="#comparison-with-alternatives" id="comparison-with-alternatives"></a>

| Feature            | Permit2        | EIP-2612         | Traditional Approve |
| ------------------ | -------------- | ---------------- | ------------------- |
| Universal Support  | Yes            | No (per-token)   | Yes                 |
| Gas for Approval   | Once per token | None (signature) | Every time          |
| Granular Control   | Excellent      | Limited          | Basic               |
| Batch Operations   | Yes            | No               | No                  |
| Signature Required | Yes            | Yes              | No                  |

### Common Issues <a href="#common-issues" id="common-issues"></a>

| Issue                     | Solution                                   |
| ------------------------- | ------------------------------------------ |
| "PERMIT\_EXPIRED"         | Increase deadline or request new signature |
| "INVALID\_SIGNATURE"      | Verify signer address and signature data   |
| "INSUFFICIENT\_ALLOWANCE" | Ensure Permit2 is approved for token       |
| "NONCE\_ALREADY\_USED"    | Use a new nonce for the permit             |

### Libraries and SDKs <a href="#libraries-and-sdks" id="libraries-and-sdks"></a>

* [Permit2 SDK](https://github.com/Uniswap/permit2-sdk) - Official TypeScript SDK
* [Permit2 Universal Router](https://github.com/Uniswap/universal-router) - Integration example
* [Permit2 Periphery](https://github.com/Uniswap/permit2-periphery) - Helper contracts

### Further Reading <a href="#further-reading" id="further-reading"></a>

* [Permit2 Introduction Blog](https://blog.uniswap.org/permit2)
* [Permit2 GitHub Repository](https://github.com/Uniswap/permit2)
* [EIP-2612: Permit Extension](https://eips.ethereum.org/EIPS/eip-2612)
* [Security Best Practices](https://github.com/Uniswap/permit2/blob/main/docs/security.md)
