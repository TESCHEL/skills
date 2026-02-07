# Foundry Project Setup

## Overview

This lesson covers installing Foundry, initializing projects, configuring foundry.toml, managing dependencies, and setting up remappings for optimal Base development workflow. Foundry provides a complete development environment with zero JavaScript dependencies.

## Installing Foundry

Foundryup is the official Foundry toolchain installer. It manages forge, cast, anvil, and chisel installations:

```bash
# Install foundryup
curl -L https://foundry.paradigm.xyz | bash

# Run foundryup to install the latest versions
foundryup

# Verify installation
forge --version
cast --version
anvil --version
chisel --version
```

Foundryup installs binaries to `~/.foundry/bin`. Add this to your PATH:

```bash
# Add to ~/.bashrc or ~/.zshrc
export PATH="$HOME/.foundry/bin:$PATH"
```

Update Foundry regularly:

```bash
foundryup
```

## Initializing a Project

Create a new Foundry project:

```bash
# Create new project with default structure
forge init my-base-project
cd my-base-project

# Or initialize in existing directory
mkdir my-project && cd my-project
forge init --force
```

Default project structure:

```
my-base-project/
  lib/              # Dependencies (git submodules)
  script/           # Deployment scripts
  src/              # Smart contracts
  test/             # Test files
  foundry.toml      # Configuration file
  .gitignore
  .gitmodules
```

The default Counter.sol example:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Counter {
    uint256 public number;

    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    function increment() public {
        number++;
    }
}
```

## Installing Dependencies

Foundry uses git submodules for dependency management. Install popular libraries:

```bash
# Install OpenZeppelin contracts
forge install OpenZeppelin/openzeppelin-contracts

# Install Solmate (gas-optimized contracts)
forge install transmissions11/solmate

# Install specific version
forge install OpenZeppelin/openzeppelin-contracts@v5.0.0

# Install from GitHub with custom path
forge install foundry-rs/forge-std --no-commit

# Update all dependencies
forge update

# Update specific dependency
forge update lib/openzeppelin-contracts

# Remove dependency
forge remove openzeppelin-contracts
```

Dependencies are stored in `lib/` as git submodules:

```bash
lib/
  forge-std/
  openzeppelin-contracts/
  solmate/
```

## Configuring foundry.toml

The foundry.toml file controls all Foundry behavior. Comprehensive Base configuration:

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.20"
evm_version = "shanghai"
optimizer = true
optimizer_runs = 200
via_ir = false

# Remappings for cleaner imports
remappings = [
    "@openzeppelin/=lib/openzeppelin-contracts/",
    "@solmate/=lib/solmate/src/",
    "forge-std/=lib/forge-std/src/"
]

# Test configuration
verbosity = 2
fuzz_runs = 256
fuzz_max_test_rejects = 65536

# Base network configuration
[rpc_endpoints]
base = "https://mainnet.base.org"
base_sepolia = "https://sepolia.base.org"

[etherscan]
base = { key = "${BASESCAN_API_KEY}" }

# Gas reporting
gas_reports = ["*"]

# Formatting
[fmt]
line_length = 100
tab_width = 4
bracket_spacing = true
int_types = "long"
multiline_func_header = "attributes_first"
quote_style = "double"
number_underscore = "thousands"

# Production profile for mainnet deployment
[profile.production]
optimizer = true
optimizer_runs = 1000000
via_ir = true

# Testing profile for comprehensive tests
[profile.test]
fuzz_runs = 10000
invariant_runs = 256

# CI profile for faster execution
[profile.ci]
fuzz_runs = 128
verbosity = 3
```

Profile-specific configurations:

```bash
# Use production profile
forge build --profile production

# Use test profile
forge test --profile test

# Set default profile
export FOUNDRY_PROFILE=production
```

## Remappings

Remappings allow clean import paths. Configure in foundry.toml or remappings.txt:

```bash
# Auto-generate remappings
forge remappings > remappings.txt
```

Example remappings.txt:

```
@openzeppelin/=lib/openzeppelin-contracts/
@solmate/=lib/solmate/src/
forge-std/=lib/forge-std/src/
ds-test/=lib/forge-std/lib/ds-test/src/
```

Using remappings in contracts:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// Clean imports using remappings
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@solmate/tokens/ERC721.sol";
import "forge-std/Test.sol";

contract MyToken is ERC20, Ownable {
    constructor() ERC20("MyToken", "MTK") Ownable(msg.sender) {}
}
```

## Environment Variables

Manage secrets and configuration with environment variables:

```bash
# Create .env file (add to .gitignore)
cat > .env << 'EOF'
PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
BASESCAN_API_KEY=your_basescan_api_key
BASE_RPC_URL=https://mainnet.base.org
BASE_SEPOLIA_RPC_URL=https://sepolia.base.org
EOF

# Load in scripts
source .env
```

Access in Solidity scripts:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";

contract DeployScript is Script {
    function run() external {
        // Read from environment
        uint256 privateKey = vm.envUint("PRIVATE_KEY");
        string memory rpcUrl = vm.envString("BASE_RPC_URL");
        
        vm.startBroadcast(privateKey);
        // Deployment logic
        vm.stopBroadcast();
    }
}
```

## Project Templates

Create custom templates for Base projects:

```bash
# Clone template
forge init --template https://github.com/username/base-template my-project

# Common Base project structure
mkdir -p my-base-project/{src,test,script,lib}
cd my-base-project

# Initialize with Base-specific contracts
cat > src/BaseRegistry.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract BaseRegistry is Ownable {
    mapping(address => string) public basenames;
    
    constructor() Ownable(msg.sender) {}
    
    function register(string calldata basename) external {
        basenames[msg.sender] = basename;
    }
}
EOF
```

## Build Process

Compile contracts:

```bash
# Build all contracts
forge build

# Build with specific solc version
forge build --use 0.8.20

# Build with optimizer
forge build --optimize --optimizer-runs 1000000

# Build and show sizes
forge build --sizes

# Clean build artifacts
forge clean

# Show build info
forge build --show-progress
```

Build output:

```
out/
  Counter.sol/
    Counter.json          # ABI and bytecode
    Counter.metadata.json # Compilation metadata
```

## Cache Management

Foundry caches compilation results:

```bash
# Clear cache
forge cache clean

# Show cache location
forge cache ls

# Disable cache
forge build --no-cache
```

## Multi-Chain Configuration

Configure multiple networks:

```toml
[rpc_endpoints]
base = "https://mainnet.base.org"
base_sepolia = "https://sepolia.base.org"
optimism = "https://mainnet.optimism.io"
arbitrum = "https://arb1.arbitrum.io"
polygon = "https://polygon-rpc.com"

[etherscan]
base = { key = "${BASESCAN_API_KEY}", url = "https://api.basescan.org/api" }
base_sepolia = { key = "${BASESCAN_API_KEY}", url = "https://api-sepolia.basescan.org/api" }
```

Use in commands:

```bash
# Deploy to Base
forge script script/Deploy.s.sol --rpc-url base --broadcast

# Deploy to Base Sepolia
forge script script/Deploy.s.sol --rpc-url base_sepolia --broadcast
```

## Common Patterns

**Pattern 1: Workspace with Multiple Projects**

```bash
mkdir base-workspace
cd base-workspace
forge init contracts
forge init --force integration-tests
```

**Pattern 2: Monorepo Structure**

```
base-monorepo/
  packages/
    core/          # Core contracts
    periphery/     # Peripheral contracts
    mocks/         # Test mocks
  foundry.toml
```

**Pattern 3: Dependency Version Pinning**

```bash
# Pin exact commit
cd lib/openzeppelin-contracts
git checkout v5.0.0
cd ../..
git add lib/openzeppelin-contracts
git commit -m "Pin OpenZeppelin to v5.0.0"
```

## Gotchas

**Gotcha 1: Remapping Conflicts**

Multiple remappings can conflict. Be explicit:

```toml
remappings = [
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
    "@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/"
]
```

**Gotcha 2: Solc Version Mismatch**

Dependencies may require different solc versions. Use via-ir for compatibility:

```toml
solc_version = "0.8.20"
via_ir = true  # Enables IR-based compilation
```

**Gotcha 3: Optimizer Settings for Base**

L2s benefit from higher optimizer runs:

```toml
optimizer_runs = 1000000  # Optimize for many executions
```

**Gotcha 4: Git Submodule Issues**

Always commit .gitmodules:

```bash
git add .gitmodules lib/
git commit -m "Add dependencies"

# Clone with submodules
git clone --recurse-submodules <repo>

# Update submodules in existing clone
git submodule update --init --recursive
```

**Gotcha 5: Environment Variable Precedence**

CLI flags override foundry.toml which overrides environment variables:

```bash
# Precedence: CLI > foundry.toml > env
forge build --optimizer-runs 200  # Highest precedence
```