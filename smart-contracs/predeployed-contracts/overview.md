# Overview

> Ready-to-use smart contracts deployed at predefined addresses

### "Pre-deployed" Contracts <a href="#andquotpre-deployedandquot-contracts" id="andquotpre-deployedandquot-contracts"></a>

#### Default Preinstalls <a href="#default-preinstalls" id="default-preinstalls"></a>

These contracts are included in `evmtypes.DefaultPreinstalls` and can be deployed at genesis or via governance:

| Contract                   | Address                                      | Purpose                                                 | Documentation |
| -------------------------- | -------------------------------------------- | ------------------------------------------------------- | ------------- |
| **Create2**                | `0x4e59b44847b379578588920ca78fbf26c0b4956c` | Deterministic contract deployment using CREATE2 opcode  | Details       |
| **Multicall3**             | `0xcA11bde05977b3631167028862bE2a173976CA11` | Batch multiple contract calls in a single transaction   | Details       |
| **Permit2**                | `0x000000000022D473030F116dDEE9F6B43aC78BA3` | Token approval and transfer management with signatures  | Details       |
| **Safe Singleton Factory** | `0x914d7Fec6aaC8cd542e72Bca78B30650d45643d7` | Deploy Safe multisig wallets at deterministic addresses | Details       |

Additional pre-deployable contracts can be incorporated into your project in a similar way, given that any dependencies are met.

### Learn More <a href="#learn-more" id="learn-more"></a>

* Implementation - Activate these contracts for your project
* Create2 - Deterministic deployment factory documentation
* Multicall3 - Batch operations contract documentation
* Permit2 - Advanced token approvals documentation
* Safe Factory - Multisig wallet factory documentation
