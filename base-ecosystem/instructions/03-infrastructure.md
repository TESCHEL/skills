# Base Technical Infrastructure

## Overview

Base is an Ethereum Layer 2 (L2) scaling solution built using the OP Stack, developed and maintained by Coinbase. As an optimistic rollup, Base executes transactions off-chain while inheriting the security guarantees of Ethereum Layer 1 (L1). Base is a core member of the Optimism Superchain, an ecosystem of interoperable L2 chains that share technology, governance, and economic alignment.

## OP Stack and Superchain Context

Base is built on the OP Stack, the open-source development framework created by Optimism. This positions Base within the Superchain ecosystem alongside other OP Stack chains including Optimism, Mode, Zora, and others. The Superchain model provides several key benefits:

- **Shared technology stack**: All Superchain members benefit from improvements to the OP Stack, including protocol upgrades, security enhancements, and performance optimizations
- **Interoperability**: Native cross-chain communication between Superchain members through shared messaging infrastructure
- **Economic alignment**: Superchain chains contribute to the growth and sustainability of the broader ecosystem
- **Standardization**: Common standards for bridges, sequencing, and data availability reduce fragmentation

By adopting the OP Stack, Base gains access to battle-tested infrastructure while contributing improvements back to the open-source codebase that benefit all Superchain participants.

## Core Architecture

### Rollup Model

Base operates as an optimistic rollup, which means:

- **Off-chain execution**: Transactions are executed on Base's L2 network, dramatically reducing costs compared to Ethereum mainnet
- **On-chain data availability**: Transaction data is posted to Ethereum L1, ensuring anyone can reconstruct the L2 state
- **Fraud proof mechanism**: Validators can challenge invalid state transitions during a 7-day challenge period
- **Ethereum settlement**: Final settlement and security are provided by Ethereum L1

This architecture enables Base to achieve high throughput and low transaction costs while maintaining strong security guarantees inherited from Ethereum.

### EVM Equivalence

Base is EVM equivalent, not merely EVM compatible. This critical distinction means:

- Base implements the Ethereum Virtual Machine specification exactly as it exists on L1
- Smart contracts can be deployed to Base without any modifications
- Development tools, libraries, and infrastructure built for Ethereum work seamlessly on Base
- Opcodes, precompiles, and gas metering behave identically to Ethereum

EVM equivalence minimizes developer friction and ensures maximum compatibility with the existing Ethereum ecosystem.

### Sequencer Architecture

Base currently operates a centralized sequencer managed by Coinbase. The sequencer is responsible for:

- Ordering transactions submitted to the network
- Executing transactions and producing blocks
- Batching transaction data for submission to Ethereum L1
- Providing instant transaction confirmations to users

**Current State**: The sequencer is operated solely by Coinbase, providing fast and reliable transaction processing.

**Decentralization Roadmap**: Coinbase and the Optimism Collective are committed to progressively decentralizing the sequencer. Future plans include:

- Multi-sequencer architecture to eliminate single points of failure
- Permissionless sequencer participation
- Revenue sharing mechanisms for sequencer operators
- Censorship resistance guarantees through forced inclusion mechanisms

Users can always bypass the sequencer by submitting transactions directly to the L1 bridge contract, ensuring censorship resistance even with a centralized sequencer.

### Data Availability

Base posts transaction data to Ethereum L1 to ensure data availability and enable independent verification of the L2 state. With the implementation of EIP-4844 (Proto-Danksharding) through the Ecotone upgrade, Base now uses blob transactions for data posting:

- **Blob transactions**: Transaction data is posted to temporary blob storage on Ethereum L1
- **Cost reduction**: Blobs are significantly cheaper than traditional calldata, reducing L1 data availability costs by approximately 10x
- **Blob lifecycle**: Blobs are available for approximately 18 days on L1, sufficient for the fraud proof window
- **Archival nodes**: Full historical data is maintained by Base archival nodes and third-party data availability services

This approach balances cost efficiency with security, ensuring data availability during the critical fraud proof window while reducing long-term storage costs.

### Fraud Proofs and Challenge Period

Base uses an optimistic fraud proof system:

- **7-day challenge period**: After state roots are proposed to L1, there is a 7-day window during which validators can submit fraud proofs
- **State verification**: Fraud proofs allow validators to challenge invalid state transitions by proving the correct execution on L1
- **Withdrawal delays**: Withdrawals from Base to Ethereum L1 require waiting for the challenge period to complete
- **Security assumption**: The system assumes at least one honest validator exists to challenge fraudulent state transitions

This optimistic approach enables high throughput while maintaining strong security guarantees through Ethereum L1 finality.

## Network Identifiers and Endpoints

### Chain IDs

- **Mainnet**: 8453
- **Sepolia Testnet**: 84532

### RPC Endpoints

**Public Endpoints**:
- Mainnet: `https://mainnet.base.org`
- Sepolia Testnet: `https://sepolia.base.org`

**Third-Party Provider Endpoints**:
- Coinbase Cloud RPC (requires API key)
- Alchemy (requires API key)
- Infura (requires API key)
- QuickNode (requires API key)
- BlockPI

Public endpoints are rate-limited. For production applications, third-party providers with dedicated infrastructure and higher rate limits are recommended.

### Block Explorer

- Mainnet: https://basescan.org
- Sepolia Testnet: https://sepolia.basescan.org

## Gas Model and Fee Structure

Base implements EIP-1559 with a two-component fee structure:

### L2 Execution Fee

- **Purpose**: Covers the cost of transaction execution on Base L2
- **Calculation**: `L2_gas_used * L2_gas_price`
- **Pricing**: Dynamic base fee that adjusts based on network congestion, following EIP-1559 mechanics
- **Target**: Blocks are produced every 2 seconds with dynamic gas limits

### L1 Data Fee

- **Purpose**: Covers the cost of posting transaction data to Ethereum L1
- **Calculation**: Based on transaction size (bytes) and current L1 gas price
- **Formula**: `transaction_size_bytes * L1_gas_price * compression_factor`
- **Blob pricing**: With EIP-4844 implementation (Ecotone upgrade), L1 data fees are calculated based on blob gas prices, reducing costs by approximately 10x compared to calldata

### Total Transaction Fee

```
Total Fee = L2 Execution Fee + L1 Data Fee
```

The L1 data fee typically dominates for simple transactions, while the L2 execution fee becomes more significant for complex smart contract interactions.

### Block Time

Base produces blocks every **2 seconds**, providing fast transaction confirmations and a responsive user experience. This is significantly faster than Ethereum L1's ~12-second block time.

## Account Abstraction Infrastructure

Base has native support for ERC-4337 account abstraction, enabling sophisticated wallet experiences and gasless transactions.

### ERC-4337 Components

**EntryPoint Contract**: Base deploys the standard ERC-4337 EntryPoint contract at `0x0576a174D229E3cFA37253523E645A78A0C91B57`, a canonical address across all EVM chains supporting account abstraction. This contract serves as the central coordinator for user operations.

**Bundler Infrastructure**: Bundlers are specialized nodes that:
- Monitor the mempool for UserOperations
- Simulate UserOperations to ensure validity
- Bundle multiple UserOperations into single transactions
- Submit bundles to the EntryPoint contract

Base is supported by major bundler providers including Alchemy, Pimlico, and Coinbase's own infrastructure.

**Paymaster Contracts**: Paymasters enable gas sponsorship and alternative fee payment methods:
- **Coinbase Paymaster**: Sponsored by Coinbase for specific use cases, enabling gasless transactions for end users
- **Custom Paymasters**: Developers can deploy custom paymaster contracts to implement application-specific gas policies
- **ERC-20 Gas Payments**: Paymasters can accept ERC-20 tokens as gas payment, abstracting away the need for users to hold ETH

### Coinbase Smart Wallet

The Coinbase Smart Wallet is an ERC-4337 smart contract wallet that leverages:

- **Passkey authentication**: Uses WebAuthn passkeys for secure, passwordless authentication
- **No seed phrases**: Eliminates the burden of seed phrase management for users
- **Gas sponsorship**: Integrates with Coinbase Paymaster for gasless transactions
- **Multi-device support**: Passkeys can be synced across devices via iCloud or Google Password Manager
- **Progressive security**: Batch transactions, spending limits, and recovery mechanisms

The Smart Wallet significantly lowers onboarding friction for mainstream users while maintaining security through modern cryptographic primitives.

## Developer Infrastructure

### OnchainKit SDK

OnchainKit is Coinbase's official SDK for building onchain applications on Base. It provides:

- **React components**: Pre-built UI components for common onchain interactions (wallet connection, transactions, NFT display)
- **TypeScript utilities**: Type-safe functions for interacting with Base and smart contracts
- **Frame support**: Tools for building Farcaster Frames with onchain functionality
- **Account abstraction integration**: Simplified APIs for working with Smart Wallets and paymasters
- **Best practices**: Opinionated patterns that follow security and UX best practices

OnchainKit accelerates development by providing production-ready components and reducing boilerplate code.

### Base SDK and Libraries

Developers can use standard Ethereum tooling:
- **ethers.js** and **viem**: For low-level RPC interactions
- **Hardhat** and **Foundry**: For smart contract development and testing
- **wagmi** and **RainbowKit**: For wallet connection and React hooks
- **The Graph**: For indexing and querying blockchain data

## Bridge Infrastructure

### Native Bridge

Base operates a native bridge at **bridge.base.org** for moving assets between Ethereum L1 and Base L2:

**Deposits (L1 to L2)**:
- Assets are locked in the L1 bridge contract
- Corresponding assets are minted on Base L2
- Deposits finalize within minutes (after L1 confirmation)

**Withdrawals (L2 to L1)**:
- Assets are burned on Base L2
- After the 7-day challenge period, assets can be claimed on L1
- The challenge period ensures security through fraud proof mechanisms

### Canonical Messaging

Base implements standard OP Stack messaging for cross-chain communication:

- **L1 to L2 messages**: Can be sent from Ethereum L1 contracts to Base L2 contracts
- **L2 to L1 messages**: Base contracts can send messages to L1, finalized after the challenge period
- **CrossDomainMessenger**: Standard contract for arbitrary message passing
- **Use cases**: Cross-chain governance, oracle updates, and multi-chain protocol coordination

## Upgradability and Governance

Base's smart contracts are upgradeable, with upgrade authority currently held by a multi-signature wallet controlled by Coinbase. The long-term vision includes:

- Progressive decentralization of upgrade authority
- Community governance participation
- Security Council mechanisms for emergency responses
- Alignment with Optimism Collective governance for Superchain-wide decisions

All upgrades follow transparent processes with advance notice to the community.

## Monitoring and Status

- **Status page**: status.base.org
- **Analytics**: base.org/metrics
- **Block explorer**: basescan.org
- **L1 data availability**: Ethereum L1 blocks contain Base transaction batches viewable on Etherscan

## Summary

Base's technical infrastructure combines the proven OP Stack with Coinbase's operational expertise to deliver a fast, cost-effective, and secure L2 scaling solution. Key highlights include:

- EVM equivalence for seamless Ethereum compatibility
- 2-second block times for responsive UX
- EIP-4844 blob transactions for 10x lower L1 data costs
- Native account abstraction support via ERC-4337
- Coinbase Smart Wallet for mainstream onboarding
- Superchain membership for ecosystem alignment and interoperability
- Commitment to progressive decentralization

This infrastructure positions Base as a leading platform for consumer-facing onchain applications that require scale, reliability, and low barriers to entry.