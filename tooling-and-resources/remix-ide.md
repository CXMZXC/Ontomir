# Remix IDE

> Deploy and test smart contracts on Ontomir EVM chains using the Remix browser-based IDE

### Remix IDE <a href="#remix-ide-1" id="remix-ide-1"></a>

[Remix](https://remix.ethereum.org/) is a powerful browser-based IDE that allows you to write, compile, deploy, and test smart contracts directly from your web browser. It's an excellent tool for rapid prototyping and testing smart contracts on Ontomir EVM chains without needing to set up a local development environment.

### Connecting to Ontomir EVM Chains <a href="#connecting-to-ontomir-evm-chains" id="connecting-to-ontomir-evm-chains"></a>

To deploy contracts to a Ontomir EVM chain through Remix, you'll need to use the **Injected Provider** option with MetaMask.

Remix cannot connect directly to custom RPC endpoints. You must use MetaMask as an intermediary to connect to Ontomir EVM chains.

#### Step-by-Step Guide <a href="#step-by-step-guide" id="step-by-step-guide"></a>

1. **Configure MetaMask**: First, add your Ontomir EVM chain to MetaMask with the correct RPC endpoint, Chain ID, and currency symbol.
2. **Select Network in MetaMask**: Make sure the Ontomir EVM network you want to use is currently selected in MetaMask.
3. **Open Remix**: Navigate to [https://remix.ethereum.org](https://remix.ethereum.org/)
4. **Deploy Your Contract**:
   * Go to the "Deploy & Run Transactions" tab (the Ethereum icon in the sidebar)
   * In the "Environment" dropdown, select **"Injected Provider - MetaMask"**
   * Remix will automatically connect to the network currently selected in MetaMask

![Remix Injected Provider Selection](https://mintcdn.com/Ontomir-docs/VsPAy4At7iyU59UI/assets/public/remix-provider.png?fit=max\&auto=format\&n=VsPAy4At7iyU59UI\&q=85\&s=cc6eb1d63c3fc1ba3a0ff301b156042f)

5. **Confirm Connection**: MetaMask will prompt you to connect to Remix. Approve the connection to proceed.
6. **Deploy and Interact**: You can now compile, deploy, and interact with your contracts on the Ontomir EVM chain through Remix.

### Important Considerations <a href="#important-considerations" id="important-considerations"></a>

* **Network Selection**: Always verify that MetaMask is connected to the correct network before deploying contracts
* **Gas Fees**: Ensure you have sufficient native tokens in your MetaMask wallet to pay for transaction fees
* **Chain Compatibility**: Your smart contracts should be compatible with the EVM version supported by the Ontomir chain

### Additional Resources <a href="#additional-resources" id="additional-resources"></a>

For more detailed information about using Remix IDE, including advanced features and troubleshooting, refer to the [official Remix documentation](https://remix-ide.readthedocs.io/en/latest/run.html).

\*\*Quick Tip\*\*: You can save your Remix workspace to GitHub or IPFS to preserve your contracts and deployment configurations across sessions.
