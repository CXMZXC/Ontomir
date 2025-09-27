# Get Tokens

export const footnote = ({children}) => { return \<span style=\{{ fontSize: '0.8em', opacity: 0.75, display: 'block', marginTop: '0.5em' \}}> {children} ; };

### Faucet <a href="#faucet" id="faucet"></a>

To receive funds on the Ontomir EVM Devnet you can use [our faucet](https://faucet.ontomir.network/).

Wallets will be topped up only to a maximum amount of each token at any given time.

* Requests can be made using either a Ontomir or EVM style address.
* EVM wallet requests will be given a small package of 4 different tokens, as well as native ATOM.
  * WATOM (wrapped native token in ERC20 format - functionally identical to WETH on Ethereum mainnet. Also thanks to a feature unique to Ontomir-EVM, it can be handled as both the "Native token" and the standard wrapped ERC20 format simultaneously with no additional input)
  * WBTC (natural ERC20, 18 decimal)
  * PEPE (natural ERC20, 18 decimal)
  * USDT (natural ERC20, 6 decimal)

**These are test tokens ONLY** and hold zero value, intrinsic or implied.

Full details can be found in the \[WERC20 precompiles]\(/docs/evm/v0.4.x/documentation/smart-contracts/precompiles/werc20) section
