---
name: viem-wagmi
description: TypeScript blockchain interaction using viem and wagmi libraries for Base
trigger_phrases:
  - "use viem to interact with contracts"
  - "set up wagmi hooks for blockchain"
  - "read contract data with viem"
  - "configure wallet client on Base"
  - "implement wagmi in React application"
version: 1.0.0
---

# Viem and Wagmi for Base Development

## Guidelines

Use the viem-wagmi skill when implementing TypeScript-based blockchain interactions on Base. Viem is a low-level TypeScript library for Ethereum-compatible chains that provides type-safe contract interactions, while Wagmi is a React Hooks library built on top of viem that simplifies wallet connections and contract operations.

### When to Use Viem

- Building Node.js backend services that interact with Base contracts
- Creating CLI tools for blockchain operations
- Implementing server-side transaction signing and submission
- Reading on-chain data without a React frontend
- Building custom blockchain indexers or monitoring tools
- When you need fine-grained control over RPC calls and transport configuration

### When to Use Wagmi

- Building React applications that require wallet connections
- Implementing user-facing DApps on Base
- Managing wallet state and account connections
- Simplifying contract reads/writes with React Hooks
- Integrating Coinbase Wallet or other Web3 wallets
- When you need automatic re-fetching and caching of blockchain data

### Best Practices

**Type Safety**: Always use TypeScript with proper ABI types. Viem provides excellent type inference from ABIs, eliminating runtime errors from incorrect function calls or parameters.

**Client Configuration**: For Base mainnet, use the built-in base chain configuration from viem/chains. Configure transports with appropriate retry logic and timeout settings for production applications.

**Error Handling**: Wrap contract interactions in try-catch blocks. Viem provides detailed error types including ContractFunctionExecutionError, TransactionExecutionError, and custom revert reasons.

**Gas Management**: Always simulate transactions with simulateContract before writing. This prevents failed transactions and wasted gas. For production, implement gas estimation with buffers.

**Event Monitoring**: Use watchContractEvent for real-time updates or getLogs for historical data. Implement proper cleanup in React useEffect hooks to prevent memory leaks.

**Multicall Optimization**: When reading multiple contract values, use multicall to batch requests into a single RPC call, reducing latency and rate limit concerns.

### Architecture Considerations

**Backend Services**: Use viem directly with createPublicClient and createWalletClient. Store private keys securely (environment variables, key management services) and never expose them client-side.

**Frontend Applications**: Use wagmi hooks exclusively. Configure WagmiConfig at the root of your React application with Base chain configuration and appropriate connectors (Coinbase Wallet, WalletConnect, etc.).

**Hybrid Applications**: Use viem for server-side operations (Next.js API routes, server components) and wagmi hooks for client-side interactions. Share ABI types and contract addresses through a common constants file.

### Common Patterns

- Reading ERC-20 balances and allowances on Base
- Executing token swaps on Aerodrome
- Interacting with Moonwell lending markets
- Querying Ethereum Attestation Service (EAS) schemas
- Resolving Basenames to addresses
- Monitoring transaction confirmations and receipts

## Examples

### Example 1: Read USDC Balance

**User Request**: "Check the USDC balance for address 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"

**Agent Action**: Use viem publicClient with readContract to call balanceOf on USDC contract (0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913), format result from 6 decimals, return balance.

### Example 2: Connect Coinbase Wallet in React

**User Request**: "Add Coinbase Wallet connection to my Base DApp"

**Agent Action**: Configure wagmi with coinbaseWallet connector, implement useConnect hook in component, provide connection button with proper error handling and account display.

### Example 3: Execute Token Swap

**User Request**: "Swap 100 USDC for ETH on Aerodrome"

**Agent Action**: Use simulateContract to validate swap parameters, call useWriteContract with Aerodrome Router (0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43), monitor transaction with useWaitForTransactionReceipt, display confirmation.

## Resources

**Viem Documentation**: https://viem.sh/docs/getting-started

**Wagmi Documentation**: https://wagmi.sh/react/getting-started

**Base Network Details**: https://docs.base.org/network-information

**Viem GitHub**: https://github.com/wevm/viem

**Wagmi GitHub**: https://github.com/wevm/wagmi

**Base Chain Configuration**: https://viem.sh/docs/chains/base

**ERC-20 ABI Reference**: https://eips.ethereum.org/EIPS/eip-20

**Contract Addresses on Base**: https://docs.base.org/docs/contracts

## Cross-References

- **typescript**: Base TypeScript configuration and best practices for blockchain development
- **nextjs**: Integrating viem and wagmi in Next.js applications with App Router
- **smart-wallet**: Using viem with ERC-4337 smart accounts and Paymaster integration
- **defi-base**: DeFi protocol interactions using viem for Aerodrome, Moonwell, and other Base protocols