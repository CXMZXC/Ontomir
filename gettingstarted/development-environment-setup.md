# Development Environment Setup

> A guide to setting up a local environment for Ontomir EVM development.

Taking the time to set up the overall development environment is often overlooked, yet is one of the most important tasks. Each person has their own preference and different tasks or scopes of work may call for vastly different setups, so this section will cover just enough to get you started, and pointed in the right direction.

### IDE Setup <a href="#ide-setup" id="ide-setup"></a>

\[Remix]\(https://remix.org) is a full-feature IDE in a web-app supporting all EVM compatible networks out of the box. A convenient option For quick testing, or as a self-contained smart contract depoyment interface. \[Read more..]\(docs/evm/next/documentation/getting-started/tooling-and-resources/remix.mdx)

#### Visual Studio / Visual Studio Code <a href="#visual-studio--visual-studio-code" id="visual-studio--visual-studio-code"></a>

"VSCode" is widely used and has unparalelled extension support.

**Essential Extensions**

* **Solidity by Nomic Foundation**: Syntax highlighting, code completion, and linting.
* **Go by Google**: Required for Ontomir SDK development.
* **Prettier - Code formatter**: For automated code formatting.
* **ESLint**: For JavaScript/TypeScript error detection.

**VS Code Configuration**

Create a `.vscode/settings.json` file in your project root:

```
{
  "go.testFlags": ["-v"],
  "go.testTimeout": "60s",
  "go.lintTool": "golangci-lint",
  "solidity.defaultCompiler": "remote",
  "solidity.compileUsingRemoteVersion": "v0.8.20+commit.a1b79de6"
}
```

### Core Tooling <a href="#core-tooling" id="core-tooling"></a>

#### Node.js <a href="#nodejs" id="nodejs"></a>

Most smart contract frameworks require Node.js. Install using Node Version Manager (nvm):

```
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash

# Install latest LTS Node.js
nvm install --lts

# Install yarn (recommended over npm)
npm install -g yarn
```

#### Go <a href="#go" id="go"></a>

Required for Ontomir SDK development:

```
# Install Go from https://go.dev/doc/install
# Verify installation
go version
```

### Environment Configuration <a href="#environment-configuration" id="environment-configuration"></a>

Configure your shell environment variables in `~/.bashrc` or `~/.zshrc`:

```
# Set Go paths
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin

# Useful aliases
alias evm-build='make build'
alias evm-test='make test'
```

### Development Workflow <a href="#development-workflow" id="development-workflow"></a>

Always use the provided \`make\` targets for consistency with core developers and CI/CD pipelines.

#### Common Commands <a href="#common-commands" id="common-commands"></a>

```
make build      # Build binaries
make test       # Run tests
make lint       # Check code style
make format     # Format code
```

#### Project Structure <a href="#project-structure" id="project-structure"></a>

```
Ontomir-evm/
├── app/           # Application configuration
├── x/             # Custom modules
│   ├── evm/       # EVM module
│   └── feemarket/ # Fee market module
├── tests/         # Integration tests
└── Makefile       # Build automation
```
