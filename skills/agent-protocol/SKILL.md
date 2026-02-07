---
name: agent-protocol
description: BAiSED Agent Protocol stack combining ERC-8004 identity, x402 payments, Circle wallets, CCTP bridging, EAS attestations, Basenames, and smart wallets for agent commerce on Base
trigger_phrases:
  - "agent protocol"
  - "agent commerce flow"
  - "discover and pay agent"
  - "agent service marketplace"
  - "HTTP 402 payment"
  - "agent payment flow"
  - "build agent marketplace"
  - "agent identity and payments"
version: 1.0.0
---

# BAiSED Agent Protocol

The BAiSED Agent Protocol is a comprehensive stack that combines all Base agent economy primitives into cohesive patterns for building autonomous agent commerce systems. It integrates ERC-8004 identity registry, HTTP 402 payment protocol, Circle programmable wallets, CCTP cross-chain bridging, EAS attestations, Basenames resolution, and ERC-4337 smart wallets into unified flows.

## Guidelines

Use this skill when building full agent commerce systems that need:

**Identity and Discovery**: Agents need verifiable on-chain identities (ERC-8004) and human-readable names (Basenames) for discovery and trust establishment. Use the identity registry for agent registration and the reputation registry for service quality tracking.

**Payment Infrastructure**: Implement HTTP 402 payment-required flows with gasless ERC-3009 USDC transfers. Circle programmable wallets enable agents to custody funds securely while smart wallets (ERC-4337) provide sponsored transactions and batched operations.

**Cross-Chain Operations**: Use CCTP for bridging USDC between Base and other chains when agents operate across multiple networks. This enables seamless cross-chain service delivery and payment settlement.

**Trust and Reputation**: EAS attestations provide verifiable service delivery proofs, capability claims, and reputation signals. Agents should both verify attestations before engaging and create attestations after service delivery.

**Composition Patterns**: The protocol defines three core patterns:
1. Agent registration + payment setup (identity + wallet creation)
2. Service discovery + attestation verification (registry query + EAS validation)
3. Cross-chain agent commerce (CCTP bridging + service delivery)

**Facilitator Architecture**: Payment facilitators act as intermediaries that:
- Present HTTP 402 payment challenges
- Verify ERC-3009 gasless transfer signatures
- Execute transfers on behalf of payers
- Distribute payments to service providers
- Take optional service fees

**Security Considerations**: Always verify agent identities through the registry, check reputation scores, validate attestations, and use nonce-based replay protection for ERC-3009 transfers. Implement rate limiting and abuse prevention in facilitator systems.

## Examples

**Example 1: Register Agent and Setup Payment**
```
User: "Register my AI agent with identity and setup payment capabilities"
Agent: Deploys ERC-4337 smart wallet, registers agent in ERC-8004 registry with metadata, creates Circle programmable wallet for USDC custody, links basename for discovery
```

**Example 2: Discover and Pay for Service**
```
User: "Find agents that can analyze smart contracts and pay one to audit my contract"
Agent: Queries registry for capability attestations, verifies EAS proofs, negotiates via HTTP 402, submits gasless USDC transfer, receives analysis, creates service delivery attestation
```

**Example 3: Cross-Chain Agent Commerce**
```
User: "Bridge funds from Ethereum and pay Base agent for data processing"
Agent: Initiates CCTP burn on Ethereum, mints on Base, discovers agent via registry, completes HTTP 402 payment flow, receives processed data, settles reputation
```

## Resources

- ERC-8004 Identity Registry: https://github.com/base-org/erc-8004
- HTTP 402 Specification: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/402
- ERC-3009 Gasless Transfers: https://eips.ethereum.org/EIPS/eip-3009
- Circle Programmable Wallets: https://developers.circle.com/w3s/programmable-wallets-quickstart
- CCTP Documentation: https://developers.circle.com/stablecoins/cctp-getting-started
- EAS Documentation: https://docs.attest.sh/docs/welcome
- ERC-4337 Spec: https://eips.ethereum.org/EIPS/eip-4337
- Basenames Documentation: https://docs.base.org/base-names/

## Cross-References

- erc-8004-registry: Agent identity and reputation foundation
- smart-wallet: ERC-4337 wallets for sponsored transactions
- cctp: Cross-chain USDC bridging for multi-chain agents
- eas-attestations: Service delivery and capability proofs
- basenames: Human-readable agent discovery
- moltbook: Intent-based coordination for complex flows
- defi-base: DeFi integration for agent treasury management
- ipfs-arweave: Decentralized storage for agent metadata and proofs
- openclaw-packaging: Standardized agent skill distribution