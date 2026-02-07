---
name: basenames
description: Register and manage .base names on Base L2 naming service
trigger_phrases:
  - "register a basename"
  - "set text records for my name"
  - "resolve basename to address"
  - "set primary name"
  - "check basename availability"
  - "update basename records"
version: 1.0.0
---

# Basenames: Base L2 Naming Service

## Guidelines

Basenames is the naming service for Base L2, providing human-readable .base names that resolve to Ethereum addresses. This skill enables you to:

- **Register Names**: Acquire .base names through the Basenames Registrar Controller at 0x4cCb0720c37C1658477b9C7A3aEBb87f1a067292
- **Manage Records**: Set and update text records (avatar, social profiles, URLs) for registered names
- **Resolve Names**: Convert .base names to addresses and vice versa using the registry at 0xb94704422c2a1e396835a571837aa5ae53285a95
- **Set Primary Names**: Configure reverse resolution so addresses display as human-readable names

Use this skill when:
- User wants to register a new .base name for their address
- User needs to set or update metadata (avatar, Twitter, GitHub, website) for their name
- User wants to resolve a .base name to an Ethereum address
- User wants to set a primary name for their address (reverse resolution)
- User needs to check availability or pricing for a basename
- User wants to transfer or manage existing basenames

Basenames integrates with the ERC-8004 Identity Registry for identity verification and Smart Wallet infrastructure for seamless account abstraction. All operations require gas on Base mainnet (chain ID 8453).

## Technical Details

**Key Contracts:**
- Registrar Controller: 0x4cCb0720c37C1658477b9C7A3aEBb87f1a067292 (registration, renewals)
- Registry: 0xb94704422c2a1e396835a571837aa5ae53285a95 (name storage, resolution)
- Reverse Registrar: Available through registry (primary name management)

**Pricing Tiers:**
- 5+ characters: ~$5 per year
- 4 characters: ~$50 per year
- 3 characters: ~$500 per year
- Prices may vary based on market conditions and length

**Name Requirements:**
- Lowercase letters, numbers, hyphens only
- 3-63 characters
- Cannot start or end with hyphen
- Must end in .base

## Examples

**Example 1: Register a New Basename**
```
User: "Register the basename alice.base for my address"
Agent: Checks availability, calculates cost for 1 year registration, submits registration transaction through Registrar Controller, confirms registration and provides name details.
```

**Example 2: Set Social Media Records**
```
User: "Set my Twitter to @alice and GitHub to alice for my basename"
Agent: Updates text records com.twitter and com.github through the registry, confirms updates, shows how records resolve.
```

**Example 3: Set Primary Name for Address**
```
User: "Make alice.base my primary name"
Agent: Configures reverse resolution through reverse registrar, sets alice.base as primary name for the address, verifies reverse lookup works.
```

## Resources

- **Basenames Documentation**: https://docs.base.org/base-names/
- **ENS Documentation**: https://docs.ens.domains/ (Basenames follows ENS standards)
- **Registry Contract**: https://basescan.org/address/0xb94704422c2a1e396835a571837aa5ae53285a95
- **Registrar Controller**: https://basescan.org/address/0x4cCb0720c37C1658477b9C7A3aEBb87f1a067292
- **Base Mainnet RPC**: https://mainnet.base.org

## Cross-References

- **erc-8004-registry**: Use identity registry (0x8004A169FB4a3325136EB29fA0ceB6D2e539a432) to link basenames with verified identities
- **smart-wallet**: Deploy smart wallets with basenames as primary identifiers for enhanced UX
- **account-abstraction**: Combine basenames with ERC-4337 (EntryPoint: 0x0576a174D229E3cFA37253523E645A78A0C91B57) for gasless name management
- **usdc-operations**: Pay for basename registrations using USDC (0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913)