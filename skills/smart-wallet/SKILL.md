---
name: smart-wallet
description: Create and manage Coinbase Smart Wallets on Base with ERC-4337 account abstraction, session keys, paymasters, and batch operations
trigger_phrases:
  - "create a smart wallet"
  - "set up session keys"
  - "use paymaster for gasless transactions"
  - "batch multiple transactions"
  - "deploy smart wallet on Base"
  - "grant temporary permissions"
  - "sponsor transaction fees"
version: 1.0.0
---

# Coinbase Smart Wallet on Base

## Guidelines

Use this skill when you need to create or interact with Coinbase Smart Wallets on Base. Smart Wallets leverage ERC-4337 account abstraction to provide enhanced user experiences including:

- Gasless transactions via paymasters
- Session keys for delegated, time-limited permissions
- Batch operations to combine multiple actions atomically
- Social recovery and multi-device access
- No seed phrase management for end users

Trigger this skill when:

1. A user wants to create a smart wallet instead of using an EOA
2. You need to enable gasless transactions for better UX
3. Implementing session-based permissions for dApps or AI agents
4. Batching multiple operations (approve + swap, multi-send, etc.)
5. Building applications that benefit from account abstraction features

Key components:
- **EntryPoint Contract**: 0x0576a174D229E3cFA37253523E645A78A0C91B57 (ERC-4337 singleton)
- **Coinbase Smart Wallet Factory**: Deterministic deployment address
- **Coinbase Paymaster**: Sponsors gas for eligible transactions
- **Session Key Plugin**: Manages delegated signing permissions

Smart Wallets are ideal for AI agents because they enable:
- Autonomous operation without exposing master keys
- Spending limits and time-bounded permissions
- Gasless execution funded by paymasters or dApp sponsors
- Complex multi-step operations in single transactions

## Examples

**Example 1: Create Smart Wallet**
- Trigger: "Create a smart wallet for me on Base"
- Action: Deploy a new Coinbase Smart Wallet using the factory contract, verify deployment, and register with ERC-8004 identity registry if desired

**Example 2: Grant Session Key**
- Trigger: "Allow this dApp to swap up to 100 USDC for the next 24 hours"
- Action: Create session key with spending limit of 100 USDC, expiration of block.timestamp + 86400, authorized for specific contract interactions

**Example 3: Batch Swap with Paymaster**
- Trigger: "Approve and swap 50 USDC for ETH using gasless transaction"
- Action: Construct UserOperation with batch calls [approve(router, 50e6), swap(50e6, path)], attach paymaster signature, submit to EntryPoint

## Resources

- ERC-4337 Specification: https://eips.ethereum.org/EIPS/eip-4337
- Coinbase Smart Wallet Docs: https://docs.cloud.coinbase.com/smartwallet/docs
- EntryPoint Contract: https://basescan.org/address/0x0576a174D229E3cFA37253523E645A78A0C91B57
- Base Account Abstraction Guide: https://docs.base.org/guides/account-abstraction
- Permissionless.js Documentation: https://docs.pimlico.io/permissionless
- Viem Account Abstraction: https://viem.sh/docs/accounts/smart

## Cross-References

- **x402**: Use X-402 protocol for authenticated smart wallet operations and capability delegation
- **erc-8004-registry**: Register smart wallet identities in the ERC-8004 registry for reputation and verification
- **cctp**: Combine smart wallet batch operations with CCTP for cross-chain USDC transfers
- **defi-base**: Use smart wallet batch operations for complex DeFi interactions on Base (Aerodrome, Moonwell)
- **basename**: Associate Basenames with smart wallet addresses for human-readable identity