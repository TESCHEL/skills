# OpenClaw Skill Format

## Overview

OpenClaw skills follow a standardized directory structure and format to enable AI agents to discover, load, and execute capabilities. This lesson covers the complete skill format specification including directory layout, YAML frontmatter, naming conventions, ASCII-only requirements, and content guidelines.

## Directory Structure

Each skill is a self-contained directory with a specific structure:

```
skills/
  skill-name/
    SKILL.md           # Required: Metadata and guidelines
    01-topic-one.md    # Lesson file (numbered)
    02-topic-two.md    # Lesson file (numbered)
    03-topic-three.md  # Lesson file (numbered)
    manifest.yaml      # Optional: Package metadata
    package.json       # Optional: npm package metadata
    examples/          # Optional: Code examples
      example1.ts
      example2.ts
    contracts/         # Optional: Smart contract ABIs
      Contract.json
```

Key requirements:
- Directory name must be kebab-case (lowercase with hyphens)
- SKILL.md is mandatory and must be uppercase
- Lesson files must start with two-digit numbers: 01-, 02-, 03-, etc.
- Lesson files should have descriptive names: 01-core-concepts.md
- Maximum 99 lesson files per skill (01-99)

## SKILL.md Format

The SKILL.md file contains YAML frontmatter followed by markdown content:

```markdown
---
name: skill-name
description: One-line description of what this skill does
trigger_phrases:
  - "phrase that triggers this skill"
  - "another trigger phrase"
  - "third trigger phrase"
version: 1.0.0
---

# Skill Title

## Guidelines

Detailed guidelines for when and how to use this skill.
Multiple paragraphs explaining the scope and context.

## Examples

**Example 1: Description**
- Trigger: "user request"
- Action: What the agent should do

**Example 2: Description**
- Trigger: "another user request"
- Action: What the agent should do

## Resources

- Link to external documentation
- Link to relevant specifications
- Link to deployed contracts

## Cross-References

- related-skill-one: Brief description
- related-skill-two: Brief description
```

## YAML Frontmatter Requirements

The frontmatter must include these exact fields:

```yaml
name: skill-directory-name  # MUST match directory name exactly
description: Single line description  # No line breaks
trigger_phrases:  # Array of strings
  - "phrase one"
  - "phrase two"
  - "phrase three"  # Minimum 3, maximum 10
version: 1.0.0  # Semantic versioning: MAJOR.MINOR.PATCH
```

Optional fields:

```yaml
author: "Author Name or Organization"
license: "MIT"  # or other SPDX identifier
tags:  # For discovery and categorization
  - "defi"
  - "base"
  - "tokens"
dependencies:  # Other skills this depends on
  - base-network-fundamentals
  - viem-ethereum-library
chain_id: 8453  # Base mainnet
contracts:  # Key contracts used by this skill
  - address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
    name: "USDC"
  - address: "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43"
    name: "Aerodrome Router"
```

## ASCII-Only Requirements

All skill content must be pure ASCII (characters 0-127). This ensures compatibility across all systems and prevents encoding issues.

**Prohibited characters:**
- Curly/smart quotes: ", ", ', ' (use " and ' instead)
- Em dash: -- (use -- double hyphen)
- En dash: - (use single hyphen -)
- Arrow symbols: ->, =>, <- (use ->, =>, <-)
- Bullet points: * (use - or *)
- Ellipsis: ... (use three periods ...)
- Emojis: Any Unicode emoji
- Accented characters: Ã©, Ã±, Ã¼ (use plain e, n, u)
- Math symbols: Ã, Ã·, â  (use *, /, !=)
- Currency symbols: â¬ (use EUR, USD text)

**Correct ASCII examples:**

```typescript
// Correct: Straight quotes
const message = "Hello, world!";
const name = 'Alice';

// Correct: Double hyphen for em dash
const note = "This is important -- very important.";

// Correct: Arrow using hyphen and greater-than
const transform = (x: number) -> number => x * 2;

// Correct: Three periods for ellipsis
const loading = "Loading...";

// Correct: Plain quotes in comments
// This function returns the user's balance
```

## Naming Conventions

**Skill directories:**
- Use kebab-case: `base-token-swaps`, `nft-marketplace`, `smart-contract-deployment`
- Be descriptive but concise: 2-4 words maximum
- Avoid abbreviations unless universally known (nft, defi, dao)
- Use singular nouns when possible: `token-swap` not `token-swaps`

**Lesson files:**
- Start with two digits: `01-`, `02-`, `03-`
- Use kebab-case for the name: `01-core-concepts.md`
- Be descriptive: `02-token-approvals-and-allowances.md`
- Order logically: Basic concepts first, advanced topics later

**Examples:**
```
skills/base-token-swaps/
  01-aerodrome-dex.md
  02-token-approvals.md
  03-slippage-protection.md
  04-multi-hop-swaps.md

skills/nft-marketplace/
  01-erc721-basics.md
  02-listing-items.md
  03-buying-and-selling.md
  04-royalty-enforcement.md
```

## Content Guidelines

**Lesson file structure:**

Each lesson file should follow this template:

```markdown
# Topic Title

## Overview

Brief description of what this lesson covers.
2-3 paragraphs maximum.

## Section One

Detailed content with code examples.

## Section Two

More detailed content.

## Common Patterns

Reusable code patterns and best practices.

## Gotchas

Common mistakes, edge cases, and warnings.
```

**Code examples:**
- Must be syntactically correct and runnable
- Include all necessary imports
- Use real contract addresses (Base mainnet)
- Prefer TypeScript with viem for Ethereum interactions
- Include error handling
- Add explanatory comments

```typescript
import { createPublicClient, http, parseUnits } from 'viem';
import { base } from 'viem/chains';

// Initialize Base client
const client = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

// USDC contract on Base
const USDC_ADDRESS = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';

// Read USDC balance
async function getUSDCBalance(address: `0x${string}`) {
  const balance = await client.readContract({
    address: USDC_ADDRESS,
    abi: [{
      name: 'balanceOf',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'account', type: 'address' }],
      outputs: [{ name: '', type: 'uint256' }]
    }],
    functionName: 'balanceOf',
    args: [address]
  });

  // USDC has 6 decimals
  return Number(balance) / 1e6;
}
```

**Contract addresses:**
- Always use real, deployed addresses
- Include chain context: "on Base" or "Base mainnet"
- Use checksummed addresses (mixed case)
- Document what the contract is

```typescript
// Core Base infrastructure
const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432';
const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63';
const ENTRYPOINT = '0x0576a174D229E3cFA37253523E645A78A0C91B57';  // ERC-4337
const USDC = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const EAS = '0x4200000000000000000000000000000000000021';  // Ethereum Attestation Service
const AERODROME_ROUTER = '0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43';
```

**Content completeness:**
- Write full, comprehensive content
- Do NOT use placeholders like "..." or "// more code here"
- Do NOT abbreviate with "etc." without examples
- Each lesson should be 80-150 lines
- SKILL.md should be 60-100 lines
- Include multiple complete code examples per lesson

**Bad example (abbreviated):**
```typescript
// Transfer tokens
const tx = await contract.transfer(...);
// ... rest of the code
```

**Good example (complete):**
```typescript
import { createWalletClient, http, parseUnits } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount('0x...');
const client = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

const USDC_ADDRESS = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';

async function transferUSDC(
  to: `0x${string}`,
  amount: number
) {
  const tx = await client.writeContract({
    address: USDC_ADDRESS,
    abi: [{
      name: 'transfer',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'to', type: 'address' },
        { name: 'amount', type: 'uint256' }
      ],
      outputs: [{ name: '', type: 'bool' }]
    }],
    functionName: 'transfer',
    args: [to, parseUnits(amount.toString(), 6)]  // USDC has 6 decimals
  });

  return tx;
}
```

## Common Patterns

**Multi-file skill organization:**

For complex skills, organize by topic progression:

```
skills/defi-lending/
  SKILL.md
  01-lending-basics.md       # Core concepts
  02-supply-assets.md        # Basic operations
  03-borrow-assets.md        # Basic operations
  04-collateral-management.md  # Intermediate
  05-liquidation-protection.md  # Advanced
  06-yield-optimization.md   # Advanced
```

**Cross-referencing:**

Always link related skills in the Cross-References section:

```markdown
## Cross-References

- base-network-fundamentals: Base chain configuration and RPC endpoints
- viem-ethereum-library: Ethereum client library used in examples
- token-standards: ERC-20, ERC-721, ERC-1155 specifications
- wallet-management: Account creation and transaction signing
```

## Gotchas

**Name mismatch:** The most common error is mismatching the directory name and YAML frontmatter name. They MUST be identical:

```
# Directory: skills/base-token-swaps/
# SKILL.md frontmatter:
name: base-token-swaps  # CORRECT

# Wrong:
name: base-token-swap   # Missing 's'
name: baseTokenSwaps    # Wrong case
name: token-swaps       # Missing 'base'
```

**Smart quotes:** Text editors and word processors often auto-convert straight quotes to curly quotes. Always verify ASCII:

```typescript
// Wrong (curly quotes)
const message = "Hello";

// Correct (straight quotes)
const message = "Hello";
```

**Incomplete code:** Never use `...` or `// more code` in examples:

```typescript
// Wrong
function doSomething() {
  // ... implementation here
}

// Correct
function doSomething() {
  const result = performCalculation();
  validateResult(result);
  return result;
}
```

**Version format:** Version must be semantic versioning with three parts:

```yaml
version: 1.0.0      # Correct
version: 1.0        # Wrong (missing patch)
version: v1.0.0     # Wrong (no 'v' prefix)
version: "1.0.0"    # Wrong (no quotes needed)
```

**File ordering:** Lesson files must have leading zeros for proper sorting:

```
01-first.md   # Correct
02-second.md  # Correct
1-first.md    # Wrong (no leading zero)
10-tenth.md   # Correct
```