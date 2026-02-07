# Viem Basics for Base Development

## Overview

Viem is a TypeScript library providing type-safe, lightweight interfaces for interacting with Ethereum-compatible blockchains like Base. Unlike legacy libraries, viem is modular, tree-shakeable, and provides excellent TypeScript inference. This lesson covers creating clients, configuring Base-specific settings, reading blockchain data, and working with encoding/decoding utilities.

## Installation and Setup

Install viem in your TypeScript project:

```bash
npm install viem
```

For Base chain configuration, also install the chains package:

```bash
npm install viem
```

Viem includes Base configuration by default in viem/chains.

## Creating Public Client

A Public Client is used for reading blockchain data without requiring a wallet or private key:

```typescript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: http("https://mainnet.base.org")
});

const blockNumber = await publicClient.getBlockNumber();
console.log("Current Base block:", blockNumber);
```

## Base Chain Configuration

The base chain object from viem/chains includes all necessary network parameters:

```typescript
import { base } from "viem/chains";

console.log("Chain ID:", base.id); // 8453
console.log("Chain Name:", base.name); // "Base"
console.log("Native Currency:", base.nativeCurrency.symbol); // "ETH"
console.log("RPC URLs:", base.rpcUrls.default.http); // ["https://mainnet.base.org"]
console.log("Block Explorer:", base.blockExplorers.default.url); // "https://basescan.org"
```

For custom RPC endpoints (Alchemy, Infura, QuickNode), override the transport:

```typescript
const publicClient = createPublicClient({
  chain: base,
  transport: http("https://base-mainnet.g.alchemy.com/v2/YOUR_API_KEY")
});
```

## Creating Wallet Client

A Wallet Client enables transaction signing and submission. Use with private keys for backend services:

```typescript
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { base } from "viem/chains";

const account = privateKeyToAccount("0xYOUR_PRIVATE_KEY");

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http("https://mainnet.base.org")
});

const hash = await walletClient.sendTransaction({
  to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  value: 1000000000000000n // 0.001 ETH in wei
});

console.log("Transaction hash:", hash);
```

## Reading Blocks and Transactions

Get current block information:

```typescript
const block = await publicClient.getBlock();
console.log("Block number:", block.number);
console.log("Block hash:", block.hash);
console.log("Timestamp:", block.timestamp);
console.log("Gas used:", block.gasUsed);
console.log("Transaction count:", block.transactions.length);
```

Get a specific block by number or hash:

```typescript
const specificBlock = await publicClient.getBlock({
  blockNumber: 10000000n
});

const blockByHash = await publicClient.getBlock({
  blockHash: "0x..."
});
```

Get transaction details:

```typescript
const transaction = await publicClient.getTransaction({
  hash: "0x4ca7ee652d57678f26e887c149ab0735f41de37bcad58c9f6d3ed5824f15b74d"
});

console.log("From:", transaction.from);
console.log("To:", transaction.to);
console.log("Value:", transaction.value);
console.log("Gas price:", transaction.gasPrice);
console.log("Input data:", transaction.input);
```

Get transaction receipt (after confirmation):

```typescript
const receipt = await publicClient.getTransactionReceipt({
  hash: "0x4ca7ee652d57678f26e887c149ab0735f41de37bcad58c9f6d3ed5824f15b74d"
});

console.log("Status:", receipt.status); // "success" or "reverted"
console.log("Block number:", receipt.blockNumber);
console.log("Gas used:", receipt.gasUsed);
console.log("Logs:", receipt.logs);
```

## Reading Account Balances

Get native ETH balance:

```typescript
const balance = await publicClient.getBalance({
  address: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
});

console.log("Balance in wei:", balance);
console.log("Balance in ETH:", Number(balance) / 1e18);
```

Format balance using viem utilities:

```typescript
import { formatEther } from "viem";

const balance = await publicClient.getBalance({
  address: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
});

console.log("Balance:", formatEther(balance), "ETH");
```

## Encoding and Decoding

Encode function data for contract calls:

```typescript
import { encodeFunctionData } from "viem";

const erc20Abi = [
  {
    name: "transfer",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "to", type: "address" },
      { name: "amount", type: "uint256" }
    ],
    outputs: [{ name: "", type: "bool" }]
  }
];

const data = encodeFunctionData({
  abi: erc20Abi,
  functionName: "transfer",
  args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", 1000000n]
});

console.log("Encoded data:", data);
```

Decode function data:

```typescript
import { decodeFunctionData } from "viem";

const decoded = decodeFunctionData({
  abi: erc20Abi,
  data: "0xa9059cbb000000000000000000000000742d35cc6634c0532925a3b844bc9e7595f0beb00000000000000000000000000000000000000000000000000000000000f4240"
});

console.log("Function name:", decoded.functionName);
console.log("Arguments:", decoded.args);
```

Encode event topics for log filtering:

```typescript
import { encodeEventTopics } from "viem";

const transferEventAbi = {
  name: "Transfer",
  type: "event",
  inputs: [
    { name: "from", type: "address", indexed: true },
    { name: "to", type: "address", indexed: true },
    { name: "value", type: "uint256", indexed: false }
  ]
};

const topics = encodeEventTopics({
  abi: [transferEventAbi],
  eventName: "Transfer",
  args: {
    from: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
  }
});

console.log("Event topics:", topics);
```

## Transport Configuration

HTTP transport with custom configuration:

```typescript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: http("https://mainnet.base.org", {
    timeout: 30000, // 30 second timeout
    retryCount: 3,
    retryDelay: 1000 // 1 second between retries
  })
});
```

Fallback transport for resilience:

```typescript
import { createPublicClient, http, fallback } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: fallback([
    http("https://mainnet.base.org"),
    http("https://base-mainnet.g.alchemy.com/v2/YOUR_API_KEY"),
    http("https://base-mainnet.infura.io/v3/YOUR_PROJECT_ID")
  ])
});
```

WebSocket transport for real-time updates:

```typescript
import { createPublicClient, webSocket } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: webSocket("wss://base-mainnet.g.alchemy.com/v2/YOUR_API_KEY")
});

const unwatch = publicClient.watchBlockNumber({
  onBlockNumber: (blockNumber) => {
    console.log("New block:", blockNumber);
  }
});

// Later: unwatch() to stop listening
```

## Working with Units

Convert between units:

```typescript
import { parseEther, parseGwei, parseUnits, formatEther, formatGwei, formatUnits } from "viem";

// Parsing (string to bigint)
const ethValue = parseEther("1.5"); // 1500000000000000000n
const gweiValue = parseGwei("20"); // 20000000000n
const usdcValue = parseUnits("100", 6); // 100000000n (USDC has 6 decimals)

// Formatting (bigint to string)
const ethString = formatEther(1500000000000000000n); // "1.5"
const gweiString = formatGwei(20000000000n); // "20"
const usdcString = formatUnits(100000000n, 6); // "100"
```

## Address Utilities

Validate and format addresses:

```typescript
import { isAddress, getAddress, isAddressEqual } from "viem";

const valid = isAddress("0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb");
console.log("Is valid address:", valid); // true

const checksummed = getAddress("0x742d35cc6634c0532925a3b844bc9e7595f0beb");
console.log("Checksummed:", checksummed); // "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"

const equal = isAddressEqual(
  "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "0x742d35cc6634c0532925a3b844bc9e7595f0beb"
);
console.log("Addresses equal:", equal); // true
```

## Common Patterns

Polling for transaction confirmation:

```typescript
async function waitForTransaction(hash: string) {
  let receipt = null;
  let attempts = 0;
  const maxAttempts = 60; // 5 minutes with 5 second intervals

  while (!receipt && attempts < maxAttempts) {
    try {
      receipt = await publicClient.getTransactionReceipt({ hash });
    } catch (error) {
      // Transaction not yet mined
    }
    
    if (!receipt) {
      await new Promise(resolve => setTimeout(resolve, 5000));
      attempts++;
    }
  }

  if (!receipt) {
    throw new Error("Transaction not confirmed after 5 minutes");
  }

  return receipt;
}
```

Batch reading multiple blocks:

```typescript
async function getBlockRange(startBlock: bigint, endBlock: bigint) {
  const blocks = [];
  
  for (let i = startBlock; i <= endBlock; i++) {
    const block = await publicClient.getBlock({ blockNumber: i });
    blocks.push(block);
  }
  
  return blocks;
}

const recentBlocks = await getBlockRange(10000000n, 10000010n);
```

## Gotchas

**BigInt Handling**: Viem uses native JavaScript BigInt for all numeric values. Never use regular numbers for token amounts or wei values. Always append 'n' suffix or use parseUnits.

**Address Checksums**: Ethereum addresses are case-sensitive for checksum validation. Always use getAddress() to ensure proper formatting before comparisons or storage.

**RPC Rate Limits**: Public RPC endpoints have rate limits. Implement exponential backoff and use fallback transports for production applications. Consider paid RPC services for reliability.

**Transaction Nonces**: When sending multiple transactions, manually manage nonces to prevent replacement issues. Get current nonce with getTransactionCount and increment for each pending transaction.

**Gas Estimation**: Base gas prices are generally low but can spike. Always fetch current gas prices with getGasPrice or use EIP-1559 fee estimation before submitting transactions.

**WebSocket Connections**: WebSocket transports require proper cleanup. Always call unwatch functions and handle reconnection logic for long-running applications.

**Type Safety**: TypeScript inference works best with const ABIs. Use 'as const' assertion on ABI objects to get full type safety on function and event parameters.