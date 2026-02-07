---
name: openclaw-packaging
description: Create, package, and publish OpenClaw AI agent skills for distribution
trigger_phrases:
  - "create a new skill package"
  - "publish skill to OpenClaw registry"
  - "package this skill for distribution"
  - "what's the skill packaging format"
  - "how do I version my skill"
  - "generate skill manifest"
version: 1.0.0
---

# OpenClaw Skill Packaging

## Guidelines

Use this skill when you need to:

- Create new skill packages for the BAiSED AI Agent Skills Library
- Understand the OpenClaw skill format and directory structure
- Generate proper SKILL.md frontmatter and metadata
- Package skills with correct naming conventions and ASCII-only formatting
- Create package manifests (package.json or manifest.yaml)
- Version skills using semantic versioning (semver)
- Publish skills to the OpenClaw registry or git repositories
- Update existing skills with version management
- Deprecate or archive outdated skills
- Set up skill dependencies and cross-references
- Tag skills for discovery and categorization

OpenClaw skills follow a strict format:
- All content must be ASCII-only (straight quotes, double hyphens, no emojis)
- Directory name must match the YAML frontmatter name exactly
- SKILL.md contains metadata and guidelines
- Lesson files (01-xxx.md, 02-xxx.md) contain detailed technical content
- All code examples must be syntactically correct and production-ready
- Contract addresses must be real, deployed addresses on Base (chain ID 8453)

Skills should be comprehensive, not abbreviated. Avoid placeholders like "..." or "// more code here". Each lesson file should be 80-150 lines with complete examples.

Versioning follows semver: MAJOR.MINOR.PATCH
- MAJOR: Breaking changes to skill interface or behavior
- MINOR: New features, additional lesson files, non-breaking enhancements
- PATCH: Bug fixes, typo corrections, documentation improvements

Skills can be distributed via:
- OpenClaw registry (centralized discovery)
- Git repositories (decentralized, version-controlled)
- IPFS (immutable, content-addressed)
- npm packages (for JavaScript/TypeScript ecosystems)

## Examples

**Example 1: Create a new skill**
- Trigger: "Create a new skill package for Base token swapping"
- Action: Generate directory structure with SKILL.md, 01-aerodrome-dex.md, 02-token-approvals.md, 03-slippage-protection.md, and manifest.yaml

**Example 2: Publish to registry**
- Trigger: "Publish the base-token-swaps skill version 1.2.0 to OpenClaw"
- Action: Validate format, generate tarball, upload to registry with metadata, update index

**Example 3: Update existing skill**
- Trigger: "Update the smart-contract-deployment skill to add Foundry support"
- Action: Add new lesson file, increment minor version to 1.3.0, update SKILL.md with new trigger phrases

## Resources

- OpenClaw Registry: https://registry.openclaw.ai
- BAiSED Skills Library: https://github.com/openclaw/baised-skills
- Semantic Versioning: https://semver.org
- YAML Spec: https://yaml.org/spec/1.2.2/
- Base Documentation: https://docs.base.org
- ASCII Character Set: https://www.ascii-code.com

## Cross-References

- agent-protocol: Understanding agent communication and skill invocation
- git: Version control for skill development and distribution
- base-network-fundamentals: Base chain IDs, RPC endpoints, contract addresses
- typescript-basics: TypeScript code examples in skills
- viem-ethereum-library: Using viem for blockchain interactions in skills