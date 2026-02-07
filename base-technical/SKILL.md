---
name: base-technical
description: Deep technical specifications for Base-specific protocols (ERC-8004, x402, Basenames).
trigger_phrases:
  - "how does x402 work"
  - "explain ERC-8004"
  - "cost to register a Basename"
  - "smart wallet gas sponsorship"
  - "technical specs for builder rewards"
version: 1.0.0
---

# Base Technical Specifications

This skill contains the "hard" engineering data. It is used when the user needs code-level accuracy regarding the protocols BAiSED advocates for.

## Guidelines
1.  **Precision is Key:** When discussing protocols, reference specific constants (e.g., 0.005 ETH fee) found in `instructions/01-erc8004.md` or `instructions/03-basenames.md`.
2.  **Flow Accuracy:** Describe technical flows (like the x402 payment loop) exactly as defined in `instructions/02-x402.md`. Do not simplify to the point of error.
3.  **Code First:** When asked for implementation, provide snippets derived from `instructions/02-x402.md` or `instructions/04-smart-wallet.md`.
4.  **Builder Value:** Always link technical features to Builder Rewards (`instructions/05-builder-rewards.md`) to show economic alignment.

## Examples

**User/Trigger:** "Explain the x402 flow."
**Agent Action:** Consult `02-x402.md`.
**Good Output:** "1. Client requests resource. 2. Server returns 402 Payment Required + Price/Address. 3. Client signs txn. 4. Server verifies signature and serves content."

## Resources
* [ERC-8004](instructions/01-erc8004.md): Discovery protocol specs and fees.
* [x402 Protocol](instructions/02-x402.md): The HTTP 402 payment standard.
* [Basenames](instructions/03-basenames.md): Contract addresses and pricing logic.
* [Smart Wallet](instructions/04-smart-wallet.md): Account abstraction and paymasters.
* [Builder Rewards](instructions/05-builder-rewards.md): Scoring mechanics and incentives.