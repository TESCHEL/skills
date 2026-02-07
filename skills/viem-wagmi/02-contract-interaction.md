# Contract Interaction with Viem

## Overview

Viem provides type-safe methods for interacting with smart contracts on Base. This lesson covers reading contract state, writing transactions, simulating calls, batching requests with multicall, watching events, and working with typed ABIs. All examples use real contract addresses deployed on Base mainnet.

## Reading Contract Data

Use readContract to call view/pure functions without gas costs:

```typescript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: http("https://mainnet.base.org")
});

const erc20Abi = [
  {
    name: "balanceOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "account", type: "address" }],
    outputs: [{ name: "", type: "uint256" }]
  },
  {
    name: "decimals",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "uint8" }]
  },
  {
    name: "symbol",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "string" }]
  }
] as const;

const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";

const balance = await publicClient.readContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "balanceOf",
  args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"]
});

const decimals = await publicClient.readContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "decimals"
});

const symbol = await publicClient.readContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "symbol"
});

console.log(`Balance: ${balance} ${symbol}`);
console.log(`Decimals: ${decimals}`);
```

## Writing Contract Transactions

Use writeContract to submit state-changing transactions:

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
  },
  {
    name: "approve",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "spender", type: "address" },
      { name: "amount", type: "uint256" }
    ],
    outputs: [{ name: "", type: "bool" }]
  }
] as const;

const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";

const hash = await walletClient.writeContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "transfer",
  args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", 1000000n] // 1 USDC (6 decimals)
});

console.log("Transaction hash:", hash);
```

Approve token spending:

```typescript
const AERODROME_ROUTER = "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43";

const approveHash = await walletClient.writeContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "approve",
  args: [AERODROME_ROUTER, 1000000000n] // 1000 USDC
});

console.log("Approval transaction:", approveHash);
```

## Simulating Contract Calls

Always simulate before writing to catch errors and estimate gas:

```typescript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: http("https://mainnet.base.org")
});

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
] as const;

try {
  const { request } = await publicClient.simulateContract({
    address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    abi: erc20Abi,
    functionName: "transfer",
    args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", 1000000n],
    account: "0xYOUR_ACCOUNT_ADDRESS"
  });

  // Simulation succeeded, now execute
  const hash = await walletClient.writeContract(request);
  console.log("Transaction submitted:", hash);
} catch (error) {
  console.error("Simulation failed:", error);
  // Handle insufficient balance, approval issues, etc.
}
```

Simulate with custom gas settings:

```typescript
const { request, result } = await publicClient.simulateContract({
  address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  abi: erc20Abi,
  functionName: "transfer",
  args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", 1000000n],
  account: "0xYOUR_ACCOUNT_ADDRESS",
  gas: 100000n,
  maxFeePerGas: 1000000000n,
  maxPriorityFeePerGas: 1000000n
});

console.log("Return value:", result);
```

## Multicall for Batch Reading

Read multiple contract values in a single RPC call:

```typescript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: http("https://mainnet.base.org")
});

const erc20Abi = [
  {
    name: "balanceOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "account", type: "address" }],
    outputs: [{ name: "", type: "uint256" }]
  },
  {
    name: "totalSupply",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "uint256" }]
  },
  {
    name: "decimals",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "uint8" }]
  },
  {
    name: "symbol",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "string" }]
  }
] as const;

const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";
const userAddress = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";

const results = await publicClient.multicall({
  contracts: [
    {
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "balanceOf",
      args: [userAddress]
    },
    {
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "totalSupply"
    },
    {
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "decimals"
    },
    {
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "symbol"
    }
  ]
});

const [balanceResult, totalSupplyResult, decimalsResult, symbolResult] = results;

if (balanceResult.status === "success") {
  console.log("Balance:", balanceResult.result);
}
if (totalSupplyResult.status === "success") {
  console.log("Total Supply:", totalSupplyResult.result);
}
if (decimalsResult.status === "success") {
  console.log("Decimals:", decimalsResult.result);
}
if (symbolResult.status === "success") {
  console.log("Symbol:", symbolResult.result);
}
```

Multicall across different contracts:

```typescript
const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";
const MOONWELL_USDC = "0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22";

const results = await publicClient.multicall({
  contracts: [
    {
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "balanceOf",
      args: [userAddress]
    },
    {
      address: MOONWELL_USDC,
      abi: erc20Abi,
      functionName: "balanceOf",
      args: [userAddress]
    }
  ]
});

console.log("USDC Balance:", results[0].result);
console.log("mUSDC Balance:", results[1].result);
```

## Event Watching and Logs

Watch for real-time contract events:

```typescript
import { createPublicClient, webSocket } from "viem";
import { base } from "viem/chains";

const publicClient = createPublicClient({
  chain: base,
  transport: webSocket("wss://base-mainnet.g.alchemy.com/v2/YOUR_API_KEY")
});

const transferEventAbi = [
  {
    name: "Transfer",
    type: "event",
    inputs: [
      { name: "from", type: "address", indexed: true },
      { name: "to", type: "address", indexed: true },
      { name: "value", type: "uint256", indexed: false }
    ]
  }
] as const;

const unwatch = publicClient.watchContractEvent({
  address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  abi: transferEventAbi,
  eventName: "Transfer",
  onLogs: (logs) => {
    logs.forEach((log) => {
      console.log("Transfer from:", log.args.from);
      console.log("Transfer to:", log.args.to);
      console.log("Amount:", log.args.value);
    });
  }
});

// Stop watching: unwatch()
```

Filter events by indexed parameters:

```typescript
const unwatch = publicClient.watchContractEvent({
  address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  abi: transferEventAbi,
  eventName: "Transfer",
  args: {
    to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
  },
  onLogs: (logs) => {
    console.log("Transfers to specific address:", logs);
  }
});
```

Get historical logs:

```typescript
const logs = await publicClient.getContractEvents({
  address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  abi: transferEventAbi,
  eventName: "Transfer",
  fromBlock: 10000000n,
  toBlock: 10001000n
});

logs.forEach((log) => {
  console.log("Block:", log.blockNumber);
  console.log("From:", log.args.from);
  console.log("To:", log.args.to);
  console.log("Value:", log.args.value);
});
```

## ABI Typing

Define ABIs with 'as const' for full type safety:

```typescript
const erc20Abi = [
  {
    name: "balanceOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "account", type: "address" }],
    outputs: [{ name: "", type: "uint256" }]
  },
  {
    name: "transfer",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "to", type: "address" },
      { name: "amount", type: "uint256" }
    ],
    outputs: [{ name: "", type: "bool" }]
  },
  {
    name: "Transfer",
    type: "event",
    inputs: [
      { name: "from", type: "address", indexed: true },
      { name: "to", type: "address", indexed: true },
      { name: "value", type: "uint256", indexed: false }
    ]
  }
] as const;

// TypeScript now infers exact types
const balance = await publicClient.readContract({
  address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  abi: erc20Abi,
  functionName: "balanceOf", // Type-checked against ABI
  args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"] // Type-checked
});

// balance is typed as bigint
console.log(balance.toString());
```

## Common ERC-20 Interaction Patterns

Check allowance before transfer:

```typescript
const erc20Abi = [
  {
    name: "allowance",
    type: "function",
    stateMutability: "view",
    inputs: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" }
    ],
    outputs: [{ name: "", type: "uint256" }]
  },
  {
    name: "approve",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "spender", type: "address" },
      { name: "amount", type: "uint256" }
    ],
    outputs: [{ name: "", type: "bool" }]
  }
] as const;

const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";
const AERODROME_ROUTER = "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43";
const ownerAddress = "0xYOUR_ADDRESS";
const amountNeeded = 1000000000n; // 1000 USDC

const currentAllowance = await publicClient.readContract({
  address: USDC_ADDRESS,
  abi: erc20Abi,
  functionName: "allowance",
  args: [ownerAddress, AERODROME_ROUTER]
});

if (currentAllowance < amountNeeded) {
  const hash = await walletClient.writeContract({
    address: USDC_ADDRESS,
    abi: erc20Abi,
    functionName: "approve",
    args: [AERODROME_ROUTER, amountNeeded]
  });
  console.log("Approval transaction:", hash);
}
```

## Common ERC-721 Interaction Patterns

```typescript
const erc721Abi = [
  {
    name: "ownerOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "tokenId", type: "uint256" }],
    outputs: [{ name: "", type: "address" }]
  },
  {
    name: "tokenURI",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "tokenId", type: "uint256" }],
    outputs: [{ name: "", type: "string" }]
  },
  {
    name: "transferFrom",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "from", type: "address" },
      { name: "to", type: "address" },
      { name: "tokenId", type: "uint256" }
    ],
    outputs: []
  }
] as const;

const owner = await publicClient.readContract({
  address: "0xYOUR_NFT_CONTRACT",
  abi: erc721Abi,
  functionName: "ownerOf",
  args: [1n]
});

console.log("Owner of token #1:", owner);

const tokenURI = await publicClient.readContract({
  address: "0xYOUR_NFT_CONTRACT",
  abi: erc721Abi,
  functionName: "tokenURI",
  args: [1n]
});

console.log("Token metadata URI:", tokenURI);
```

## Interacting with Basenames

Resolve Basename to address:

```typescript
const basenamesRegistryAbi = [
  {
    name: "resolver",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "node", type: "bytes32" }],
    outputs: [{ name: "", type: "address" }]
  }
] as const;

const resolverAbi = [
  {
    name: "addr",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "node", type: "bytes32" }],
    outputs: [{ name: "", type: "address" }]
  }
] as const;

import { namehash } from "viem";

const BASENAMES_REGISTRY = "0xb94704422c2a1e396835a571837aa5ae53285a95";
const name = "alice.base.eth";
const node = namehash(name);

const resolverAddress = await publicClient.readContract({
  address: BASENAMES_REGISTRY,
  abi: basenamesRegistryAbi,
  functionName: "resolver",
  args: [node]
});

const address = await publicClient.readContract({
  address: resolverAddress,
  abi: resolverAbi,
  functionName: "addr",
  args: [node]
});

console.log(`${name} resolves to:`, address);
```

## Gotchas

**ABI Type Safety**: Always use 'as const' on ABI definitions. Without it, TypeScript cannot infer exact function names and parameter types, losing viem's main advantage.

**Simulation Before Writing**: Never skip simulateContract in production. It catches 90% of transaction failures before spending gas, including insufficient balance, approval issues, and reverts.

**Event Filtering**: When watching events, be specific with indexed parameters. Watching all Transfer events on USDC will flood your application with data. Filter by relevant addresses.

**Multicall Failures**: In multicall, individual calls can fail without affecting others. Always check the status field of each result before accessing the result property.

**Gas Estimation**: simulateContract provides gas estimates, but add a 20-30% buffer for actual execution. Network conditions and contract state changes can affect gas usage.

**WebSocket Cleanup**: Always store the unwatch function returned by watchContractEvent and call it when unmounting components or stopping services to prevent memory leaks.

**Large Number Handling**: All numeric values are bigint. Never use Number() for token amounts or you'll lose precision. Use formatUnits for display and parseUnits for input.

**Transaction Ordering**: When approving and then spending tokens, wait for approval confirmation before submitting the spend transaction. Use waitForTransactionReceipt between calls.