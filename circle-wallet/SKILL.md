---
name: circle-wallet
description: Integration guide for Circle's Programmable Wallets and USDC infrastructure on Base.
trigger_phrases:
  - "create a circle wallet"
  - "integrate circle programmable wallets"
  - "use USDC with circle"
  - "set up developer-controlled wallets"
  - "circle wallet SDK"
version: 1.0.0
---

# Circle Wallet Integration

This skill enables the agent to understand and implement Circle's Programmable Wallets SDK for creating and managing wallets on Base. Circle provides the infrastructure for USDC and developer-controlled or user-controlled wallets.

## Guidelines
1.  **Wallet Types:** Understand the distinction between developer-controlled wallets (server-side, no user interaction needed) and user-controlled wallets (requires user PIN/biometrics).
2.  **USDC Native:** Circle wallets are optimized for USDC transactions. USDC on Base is at `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`.
3.  **API Key Security:** Never expose Circle API keys in client-side code. All API calls should be server-side.
4.  **Idempotency:** Use idempotency keys for all wallet creation and transfer operations to prevent duplicates.

## Examples

**User/Trigger:** "How do I create a developer-controlled wallet on Base?"
**Agent Action:** Consult instructions/01-setup.md.
**Good Output:** "Use the Circle SDK with your API key. Call createWalletSet() first, then createWallet() with the wallet set ID and Base blockchain."

**User/Trigger:** "How do I send USDC from a Circle wallet?"
**Agent Action:** Consult instructions for transfer operations.
**Good Output:** "Use the createTransaction() method with the source wallet ID, destination address, amount in USDC (6 decimals), and the Base chain."

## Resources
* [Setup Guide](instructions/01-setup.md): SDK installation and API key configuration.
