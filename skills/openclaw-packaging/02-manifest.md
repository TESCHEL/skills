# Package Manifest

## Overview

OpenClaw skills use manifest files to declare metadata, dependencies, versioning, and discovery tags. This lesson covers both package.json (npm-style) and manifest.yaml (OpenClaw-native) formats, semantic versioning, dependency management, and skill discovery mechanisms.

## Manifest Formats

OpenClaw supports two manifest formats:

**manifest.yaml (recommended):**
```yaml
name: base-token-swaps
version: 1.2.0
description: Swap tokens on Base using Aerodrome DEX
author: OpenClaw Community
license: MIT
repository: https://github.com/openclaw/baised-skills

chain:
  id: 8453
  name: Base
  rpc: https://mainnet.base.org

contracts:
  - address: "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43"
    name: Aerodrome Router
    abi: ./contracts/AerodromeRouter.json
  - address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
    name: USDC
    abi: ./contracts/ERC20.json

dependencies:
  base-network-fundamentals: ^1.0.0
  viem-ethereum-library: ^2.0.0
  token-standards: ^1.1.0

tags:
  - defi
  - dex
  - tokens
  - swaps
  - aerodrome
  - base

files:
  - SKILL.md
  - 01-aerodrome-dex.md
  - 02-token-approvals.md
  - 03-slippage-protection.md
  - 04-multi-hop-swaps.md

engines:
  openclaw: ">=1.0.0"
  node: ">=18.0.0"
```

**package.json (npm-compatible):**
```json
{
  "name": "@openclaw/base-token-swaps",
  "version": "1.2.0",
  "description": "Swap tokens on Base using Aerodrome DEX",
  "author": "OpenClaw Community",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/openclaw/baised-skills"
  },
  "keywords": [
    "openclaw",
    "skill",
    "defi",
    "dex",
    "tokens",
    "swaps",
    "aerodrome",
    "base"
  ],
  "openclaw": {
    "skill": true,
    "chain": {
      "id": 8453,
      "name": "Base",
      "rpc": "https://mainnet.base.org"
    },
    "contracts": [
      {
        "address": "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43",
        "name": "Aerodrome Router",
        "abi": "./contracts/AerodromeRouter.json"
      },
      {
        "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "name": "USDC",
        "abi": "./contracts/ERC20.json"
      }
    ]
  },
  "dependencies": {
    "@openclaw/base-network-fundamentals": "^1.0.0",
    "@openclaw/viem-ethereum-library": "^2.0.0",
    "@openclaw/token-standards": "^1.1.0"
  },
  "engines": {
    "openclaw": ">=1.0.0",
    "node": ">=18.0.0"
  },
  "files": [
    "SKILL.md",
    "01-aerodrome-dex.md",
    "02-token-approvals.md",
    "03-slippage-protection.md",
    "04-multi-hop-swaps.md",
    "contracts/"
  ]
}
```

## Semantic Versioning

OpenClaw skills follow semantic versioning (semver): MAJOR.MINOR.PATCH

**Version components:**
- MAJOR: Incompatible changes that break existing usage
- MINOR: New features that are backward-compatible
- PATCH: Bug fixes and documentation improvements

**Version increment rules:**

```yaml
# Initial release
version: 1.0.0

# Add new lesson file -> increment MINOR
version: 1.1.0

# Fix typo in existing lesson -> increment PATCH
version: 1.1.1

# Change skill interface or trigger phrases -> increment MAJOR
version: 2.0.0

# Add new trigger phrase (backward-compatible) -> increment MINOR
version: 2.1.0
```

**Pre-release versions:**

```yaml
# Alpha release
version: 1.0.0-alpha.1

# Beta release
version: 1.0.0-beta.1

# Release candidate
version: 1.0.0-rc.1

# Final release
version: 1.0.0
```

**Version ranges in dependencies:**

```yaml
dependencies:
  # Exact version
  base-network-fundamentals: 1.0.0
  
  # Compatible with 1.x.x (>=1.0.0 <2.0.0)
  viem-ethereum-library: ^1.0.0
  
  # Compatible with 1.2.x (>=1.2.0 <1.3.0)
  token-standards: ~1.2.0
  
  # Any version >=2.0.0
  wallet-management: ">=2.0.0"
  
  # Version range
  defi-protocols: ">=1.0.0 <3.0.0"
```

## Metadata Fields

**Required fields:**

```yaml
name: skill-name              # Must match directory name
version: 1.0.0                # Semantic version
description: Short description  # One line, <100 characters
```

**Recommended fields:**

```yaml
author: Author Name or Org
license: MIT                  # SPDX identifier
repository: https://github.com/org/repo
homepage: https://example.com
bugs: https://github.com/org/repo/issues
```

**OpenClaw-specific fields:**

```yaml
chain:
  id: 8453                    # Chain ID (Base mainnet)
  name: Base
  rpc: https://mainnet.base.org
  explorer: https://basescan.org

contracts:
  - address: "0x..."
    name: Contract Name
    abi: ./path/to/abi.json
    verified: true            # Whether source is verified on explorer

tags:
  - primary-category          # First tag is primary
  - secondary-category
  - feature-keyword

engines:
  openclaw: ">=1.0.0"         # Minimum OpenClaw version
  node: ">=18.0.0"            # Minimum Node.js version
```

**Extended metadata:**

```yaml
maintainers:
  - name: Alice
    email: alice@example.com
    github: alice
  - name: Bob
    email: bob@example.com
    github: bob

contributors:
  - name: Charlie
    github: charlie

sponsor:
  type: individual
  url: https://github.com/sponsors/alice

funding:
  - type: opencollective
    url: https://opencollective.com/openclaw
  - type: github
    url: https://github.com/sponsors/openclaw
```

## Contract Metadata

Contract declarations help agents understand which contracts the skill interacts with:

```yaml
contracts:
  # ERC-20 Token
  - address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
    name: USDC
    type: ERC20
    abi: ./contracts/ERC20.json
    decimals: 6
    verified: true
    
  # DEX Router
  - address: "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43"
    name: Aerodrome Router
    type: UniswapV2Router
    abi: ./contracts/AerodromeRouter.json
    verified: true
    docs: https://aerodrome.finance/docs
    
  # NFT Contract
  - address: "0x1234567890123456789012345678901234567890"
    name: Base Apes NFT
    type: ERC721
    abi: ./contracts/ERC721.json
    verified: true
    
  # Custom Protocol
  - address: "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432"
    name: Identity Registry
    type: Custom
    abi: ./contracts/IdentityRegistry.json
    verified: true
    docs: https://docs.openclaw.ai/identity
```

## Dependency Management

Skills can depend on other skills to reuse knowledge and functionality:

```yaml
dependencies:
  # Core dependencies
  base-network-fundamentals: ^1.0.0  # Chain config, RPC setup
  viem-ethereum-library: ^2.0.0      # Ethereum interactions
  
  # Feature dependencies
  token-standards: ^1.1.0            # ERC-20, ERC-721 knowledge
  wallet-management: ^1.0.0          # Account handling
  
  # Protocol dependencies
  aerodrome-dex: ^1.2.0              # DEX-specific knowledge
  
# Optional dependencies (not required but enhance functionality)
optionalDependencies:
  ens-resolution: ^1.0.0             # ENS name resolution
  gasless-transactions: ^1.0.0       # ERC-4337 integration
```

**Dependency resolution:**

When an agent loads a skill, it recursively loads dependencies:

```typescript
// Skill loader pseudocode
async function loadSkill(skillName: string): Promise<Skill> {
  const manifest = await loadManifest(skillName);
  
  // Load dependencies first
  const dependencies = await Promise.all(
    Object.keys(manifest.dependencies).map(dep => loadSkill(dep))
  );
  
  // Load skill content
  const skill = await parseSkillContent(skillName);
  
  // Merge knowledge from dependencies
  skill.knowledge = mergeDependencies(dependencies, skill);
  
  return skill;
}
```

**Circular dependencies:**

Avoid circular dependencies. If skill A depends on B, B cannot depend on A:

```yaml
# skills/token-swaps/manifest.yaml
dependencies:
  token-approvals: ^1.0.0  # OK

# skills/token-approvals/manifest.yaml
dependencies:
  token-swaps: ^1.0.0      # ERROR: Circular dependency
```

## Discovery Tags

Tags enable skill discovery and categorization:

```yaml
tags:
  # Primary category (first tag)
  - defi
  
  # Protocol/Platform
  - base
  - aerodrome
  - uniswap
  
  # Feature/Capability
  - swaps
  - tokens
  - liquidity
  - yield
  
  # Token types
  - erc20
  - stablecoins
  
  # Complexity level
  - beginner
  - intermediate
  - advanced
  
  # Use case
  - trading
  - portfolio
  - automation
```

**Standard tag categories:**

```yaml
# Blockchain/Network
- base
- ethereum
- optimism
- arbitrum

# Protocol type
- defi
- nft
- dao
- identity
- gaming

# DeFi categories
- dex
- lending
- staking
- yield
- liquidity

# NFT categories
- marketplace
- minting
- collections
- metadata

# Infrastructure
- oracles
- bridges
- l2
- account-abstraction

# Development
- smart-contracts
- testing
- deployment
- verification

# Standards
- erc20
- erc721
- erc1155
- erc4337
```

## Files Declaration

Explicitly list files included in the package:

```yaml
files:
  # Core skill files
  - SKILL.md
  - "*.md"              # All markdown files
  - manifest.yaml
  
  # Contract ABIs
  - contracts/
  - contracts/*.json
  
  # Code examples
  - examples/
  - examples/**/*.ts
  
  # Documentation
  - README.md
  - CHANGELOG.md
  - LICENSE
```

**Exclusion patterns:**

```yaml
# .openclawignore file
node_modules/
.git/
.env
*.log
test/
*.test.ts
.DS_Store
thumbs.db
```

## Engine Requirements

Specify minimum versions for OpenClaw and runtime:

```yaml
engines:
  openclaw: ">=1.0.0"     # Minimum OpenClaw version
  node: ">=18.0.0"        # Minimum Node.js version
  npm: ">=9.0.0"          # Minimum npm version
```

**Version checking:**

```typescript
import { satisfies } from 'semver';

function checkEngines(manifest: Manifest): boolean {
  const currentOpenClaw = '1.2.0';
  const currentNode = process.version;
  
  if (!satisfies(currentOpenClaw, manifest.engines.openclaw)) {
    throw new Error(
      `OpenClaw ${manifest.engines.openclaw} required, ` +
      `found ${currentOpenClaw}`
    );
  }
  
  if (!satisfies(currentNode, manifest.engines.node)) {
    throw new Error(
      `Node.js ${manifest.engines.node} required, ` +
      `found ${currentNode}`
    );
  }
  
  return true;
}
```

## Common Patterns

**Multi-chain skill manifest:**

```yaml
name: cross-chain-bridge
version: 1.0.0
description: Bridge tokens across multiple chains

chains:
  - id: 8453
    name: Base
    rpc: https://mainnet.base.org
    contracts:
      - address: "0x1682Ae6375C4E4A97e4B583BC394c861A46D8962"
        name: CCTP TokenMessenger
  - id: 10
    name: Optimism
    rpc: https://mainnet.optimism.io
    contracts:
      - address: "0x..."
        name: CCTP TokenMessenger

tags:
  - bridge
  - cross-chain
  - cctp
  - base
  - optimism
```

**Protocol suite manifest:**

```yaml
name: aerodrome-protocol
version: 2.0.0
description: Complete Aerodrome DEX protocol suite

contracts:
  - address: "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43"
    name: Router
    type: Router
  - address: "0x..."
    name: Factory
    type: Factory
  - address: "0x..."
    name: Voter
    type: Governance

subskills:
  - aerodrome-swaps
  - aerodrome-liquidity
  - aerodrome-voting
  - aerodrome-gauges

tags:
  - defi
  - dex
  - aerodrome
  - liquidity
  - governance
```

## Gotchas

**Name consistency:** Package name in manifest must match directory name:

```yaml
# Directory: skills/base-token-swaps/
# manifest.yaml
name: base-token-swaps    # CORRECT

# package.json (npm scope optional)
"name": "@openclaw/base-token-swaps"  # CORRECT
"name": "base-token-swaps"            # Also correct
```

**Version as string:** YAML versions should be strings to preserve format:

```yaml
version: "1.0.0"    # Correct (string)
version: 1.0.0      # Also works but may lose trailing zeros
version: 1.0        # Wrong (will be parsed as number 1.0)
```

**Tag ordering:** First tag is considered the primary category:

```yaml
tags:
  - defi        # Primary category
  - base
  - tokens
```

**Contract addresses:** Always use checksummed addresses:

```yaml
contracts:
  - address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"  # Correct
  - address: "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913"  # Wrong (lowercase)
```

**Dependency versions:** Use caret (^) for flexible minor updates:

```yaml
dependencies:
  base-network: ^1.0.0    # Allows 1.x.x
  base-network: ~1.0.0    # Allows 1.0.x only
  base-network: 1.0.0     # Exact version (too strict)
```