---
name: Base Technical Reference
description: Deep technical specifications for Base L2 infrastructure â contract addresses, function signatures, payment flows, and implementation details. Use for building onchain integrations, not content strategy.
trigger_phrases:
  - "ERC-8004 registration"
  - "x402 payment flow"
  - "Basename registration"
  - "smart wallet setup"
  - "builder rewards"
  - "contract addresses"
  - "account abstraction"
version: 1.0.0
dependencies:
  - base-ecosystem
---

# Base Technical Reference

Implementation-level documentation for Base L2 infrastructure. This skill contains contract addresses, function signatures, SDK references, and step-by-step flows for building onchain.

## When to use this skill
- Registering BAiSED as an onchain agent (ERC-8004)
- Implementing or discussing x402 micropayments
- Registering or managing Basenames
- Setting up smart wallets and session keys
- Qualifying for Builder Rewards and grants

## Instruction files
- `01-erc8004.md` â Agent Directory: registration, reputation, contract addresses
- `02-x402.md` â HTTP 402 micropayment protocol: full payment flow, headers, code
- `03-basenames.md` â Programmatic name registration, pricing, text records
- `04-smart-wallet.md` â Account abstraction, sub accounts, session keys, paymaster
- `05-builder-rewards.md` â Builder Score, grants, Base Batches, RetroPGF

## Relationship to other skills
- `base-ecosystem/` = narrative context (what Base is, who matters, recent developments)
- `base-technical/` = implementation details (how to build on Base)
- `content-strategy/` = how to talk about it