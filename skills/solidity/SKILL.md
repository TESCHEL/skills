---
name: solidity
description: Smart contract development with Solidity on Base (Coinbase L2)
trigger_phrases:
  - "write a solidity contract"
  - "deploy smart contract to base"
  - "solidity development on base"
  - "create erc20 token"
  - "implement smart contract"
version: 1.0.0
---

# Solidity Smart Contract Development on Base

## Guidelines

This skill covers writing, deploying, and managing Solidity smart contracts on Base (Coinbase L2). Use this skill when agents need to:

- Write new Solidity contracts from scratch
- Understand Solidity language features and best practices
- Implement token standards (ERC-20, ERC-721, ERC-1155)
- Apply Base-specific optimization patterns for L2 deployment
- Implement security patterns and access control
- Work with proxy patterns and upgradeable contracts
- Integrate with Base system contracts and precompiles
- Optimize gas usage for Layer 2 execution

Solidity version 0.8.x is recommended for all Base deployments due to built-in overflow protection and improved error handling. Base is fully EVM-compatible, so standard Solidity patterns work, but L2-specific optimizations can significantly reduce transaction costs.

Key considerations for Base:
- Gas costs are lower than Ethereum mainnet but still matter
- L1->L2 messaging patterns for cross-chain functionality
- Base system contracts at predictable addresses
- Faster block times (2 seconds) affect time-dependent logic
- Use CREATE2 for deterministic deployment addresses

## Examples

**Example 1: Creating an ERC-20 Token**

Trigger: "Create a standard ERC-20 token contract for Base"

Action: Generate a Solidity contract implementing ERC-20 with OpenZeppelin base contracts, including minting, burning, and access control. Provide deployment instructions using Foundry or Hardhat targeting Base mainnet (chain ID 8453).

**Example 2: Implementing Upgradeable Contract**

Trigger: "Write an upgradeable contract using UUPS proxy pattern"

Action: Create a UUPS proxy implementation with initialization function, upgrade authorization, and storage layout documentation. Include deployment script for both proxy and implementation contracts.

**Example 3: Base-Optimized Contract**

Trigger: "Optimize this contract for Base L2 deployment"

Action: Review contract for gas optimization opportunities specific to L2: minimize storage writes, use events for data availability, leverage calldata over memory where appropriate, and implement efficient batch operations.

## Resources

- Solidity Documentation: https://docs.soliditylang.org/
- Base Developer Documentation: https://docs.base.org/
- OpenZeppelin Contracts: https://docs.openzeppelin.com/contracts/
- Base Contract Addresses: https://docs.base.org/base-contracts
- Solidity Style Guide: https://docs.soliditylang.org/en/latest/style-guide.html
- Base Mainnet RPC: https://mainnet.base.org
- Base Chain ID: 8453

## Cross-References

- foundry: Compile, test, and deploy Solidity contracts using Foundry toolkit
- hardhat: Alternative development environment for Solidity projects
- viem-wagmi: Interact with deployed contracts from TypeScript/frontend
- erc-8004-registry: Identity and reputation registries on Base
- account-abstraction: ERC-4337 smart account patterns
- cross-chain: CCTP and messaging for cross-chain Solidity contracts