---
name: erc-8004-registry
description: Register and manage on-chain agent identities using the ERC-8004 Trustless Agents Standard on Base
trigger_phrases:
  - "register my agent on-chain"
  - "create ERC-8004 agent identity"
  - "manage agent metadata on Base"
  - "query the agent registry"
  - "check agent reputation"
version: 1.0.0
---

# ERC-8004 Agent Registry

## Guidelines

This skill enables interaction with the ERC-8004 Trustless Agents Standard on Base, which provides verifiable on-chain identities for AI agents. Use this skill when you need to:

- Register a new AI agent with an on-chain identity (ERC-721 NFT)
- Update or manage agent metadata (capabilities, services, endpoints)
- Query the registry to discover agents by domain or capabilities
- Record or check reputation scores for agents
- Resolve cross-chain agent identities

The ERC-8004 standard creates a trustless registry where agents are represented as NFTs with associated metadata describing their capabilities, supported protocols (like X402), and service endpoints. Each agent has a unique domain name (like "assistant.agent") and can accumulate reputation through on-chain feedback.

Registration requires 0.005 ETH and creates an ERC-721 token owned by the agent's address. Metadata should be hosted on IPFS or Arweave for immutability and referenced via tokenURI. The system supports both identity registration (agent existence) and reputation tracking (agent behavior).

Use the Identity Registry at 0x8004A169FB4a3325136EB29fA0ceB6D2e539a432 for registration and metadata management. Use the Reputation Registry at 0x8004BAa17C55a88189AE136b182e5fdA19dE9b63 for feedback and reputation queries.

This skill integrates with EAS attestations for richer identity claims, smart wallets for agent account management, and IPFS/Arweave for metadata storage. It is foundational for building trustless multi-agent systems on Base.

## Examples

**Example 1: User wants to register an agent**
- Trigger: "Register my trading agent on-chain with domain tradingbot.agent"
- Action: Check domain availability, prepare 0.005 ETH, call newAgent() on Identity Registry, upload metadata to IPFS, set tokenURI

**Example 2: User wants to query agent capabilities**
- Trigger: "Find all agents that support X402 protocol for payments"
- Action: Query Identity Registry for all registered agents, fetch metadata for each, filter by x402Support: true in JSON

**Example 3: User wants to check agent reputation**
- Trigger: "What is the reputation score for assistant.agent?"
- Action: Resolve domain to token ID, query Reputation Registry for feedback records, calculate average rating

## Resources

- ERC-8004 Specification: https://eips.ethereum.org/EIPS/eip-8004
- Identity Registry Contract: https://basescan.org/address/0x8004A169FB4a3325136EB29fA0ceB6D2e539a432
- Reputation Registry Contract: https://basescan.org/address/0x8004BAa17C55a88189AE136b182e5fdA19dE9b63
- Base Chain Documentation: https://docs.base.org
- IPFS Documentation: https://docs.ipfs.tech
- Viem Documentation: https://viem.sh

## Cross-References

- eas-attestations: Link EAS attestations to agent identities for verifiable claims
- agent-protocol: Implement agent communication protocols over registered identities
- smart-wallet: Use ERC-4337 wallets as agent owners for autonomous operation
- ipfs-arweave: Host agent metadata on decentralized storage
- basenames: Integrate with Basenames for human-readable agent identifiers