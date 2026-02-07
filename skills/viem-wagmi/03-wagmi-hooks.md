# Wagmi Hooks for React Applications

## Overview

Wagmi is a React Hooks library built on viem that simplifies wallet connections and blockchain interactions in React applications. It provides automatic connection management, transaction state handling, and data caching. This lesson covers essential hooks, Coinbase Wallet integration, and configuring wagmi for Base.

## Installation and Setup

Install wagmi and dependencies:

```bash
npm install wagmi viem@2.x @tanstack/react-query
```

Install Coinbase Wallet connector:

```bash
npm install @coinbase/wallet-sdk
```

## Wagmi Configuration for Base

Create wagmi config at application root:

```typescript
import { http, createConfig } from "wagmi";
import { base } from "wagmi/chains";
import { coinbaseWallet } from "wagmi/connectors";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const config = createConfig({
  chains: [base],
  connectors: [
    coinbaseWallet({
      appName: "My Base DApp",
      appLogoUrl: "https://example.com/logo.png",
      preference: "smartWalletOnly" // or "all" for both smart and EOA
    })
  ],
  transports: {
    [base.id]: http("https://mainnet.base.org")
  }
});

const queryClient = new QueryClient();

export { config, queryClient };
```

Wrap application with providers:

```typescript
import { WagmiProvider } from "wagmi";
import { QueryClientProvider } from "@tanstack/react-query";
import { config, queryClient } from "./config";

function App() {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        {/* Your app components */}
      </QueryClientProvider>
    </WagmiProvider>
  );
}

export default App;
```

## useAccount Hook

Access connected account information:

```typescript
import { useAccount } from "wagmi";

function AccountInfo() {
  const { address, isConnected, isConnecting, isDisconnected, chain } = useAccount();

  if (isConnecting) {
    return <div>Connecting wallet...</div>;
  }

  if (isDisconnected) {
    return <div>No wallet connected</div>;
  }

  return (
    <div>
      <p>Connected Address: {address}</p>
      <p>Chain: {chain?.name} (ID: {chain?.id})</p>
      <p>Status: {isConnected ? "Connected" : "Disconnected"}</p>
    </div>
  );
}
```

Display shortened address:

```typescript
import { useAccount } from "wagmi";

function WalletButton() {
  const { address, isConnected } = useAccount();

  const shortenAddress = (addr: string) => {
    return `${addr.slice(0, 6)}...${addr.slice(-4)}`;
  };

  if (!isConnected) {
    return <button>Connect Wallet</button>;
  }

  return (
    <div>
      <span>{shortenAddress(address!)}</span>
      <button>Disconnect</button>
    </div>
  );
}
```

## useConnect Hook

Manage wallet connections:

```typescript
import { useConnect, useAccount } from "wagmi";

function ConnectWallet() {
  const { connectors, connect, status, error } = useConnect();
  const { isConnected } = useAccount();

  if (isConnected) {
    return <div>Wallet connected!</div>;
  }

  return (
    <div>
      <h2>Connect Your Wallet</h2>
      {connectors.map((connector) => (
        <button
          key={connector.id}
          onClick={() => connect({ connector })}
          disabled={status === "pending"}
        >
          {connector.name}
        </button>
      ))}
      {error && <div>Error: {error.message}</div>}
    </div>
  );
}
```

Connect with Coinbase Wallet specifically:

```typescript
import { useConnect } from "wagmi";
import { coinbaseWallet } from "wagmi/connectors";

function CoinbaseWalletConnect() {
  const { connect, status, error } = useConnect();

  const handleConnect = () => {
    connect({
      connector: coinbaseWallet({
        appName: "My Base DApp",
        preference: "smartWalletOnly"
      })
    });
  };

  return (
    <div>
      <button onClick={handleConnect} disabled={status === "pending"}>
        {status === "pending" ? "Connecting..." : "Connect Coinbase Wallet"}
      </button>
      {error && <p>Error: {error.message}</p>}
    </div>
  );
}
```

## useDisconnect Hook

Disconnect wallet:

```typescript
import { useDisconnect, useAccount } from "wagmi";

function DisconnectButton() {
  const { disconnect } = useDisconnect();
  const { isConnected } = useAccount();

  if (!isConnected) {
    return null;
  }

  return (
    <button onClick={() => disconnect()}>
      Disconnect Wallet
    </button>
  );
}
```

## useReadContract Hook

Read contract data with automatic caching and refetching:

```typescript
import { useReadContract } from "wagmi";
import { formatUnits } from "viem";

const erc20Abi = [
  {
    name: "balanceOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "account", type: "address" }],
    outputs: [{ name: "", type: "uint256" }]
  },
  {
    name: "symbol",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ name: "", type: "string" }]
  }
] as const;

function USDCBalance({ address }: { address: string }) {
  const { data: balance, isLoading, isError, error } = useReadContract({
    address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    abi: erc20Abi,
    functionName: "balanceOf",
    args: [address as `0x${string}`]
  });

  const { data: symbol } = useReadContract({
    address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    abi: erc20Abi,
    functionName: "symbol"
  });

  if (isLoading) return <div>Loading balance...</div>;
  if (isError) return <div>Error: {error?.message}</div>;

  return (
    <div>
      Balance: {formatUnits(balance || 0n, 6)} {symbol}
    </div>
  );
}
```

Read with watch option for automatic updates:

```typescript
function LiveBalance({ address }: { address: string }) {
  const { data: balance } = useReadContract({
    address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    abi: erc20Abi,
    functionName: "balanceOf",
    args: [address as `0x${string}`],
    query: {
      refetchInterval: 10000 // Refetch every 10 seconds
    }
  });

  return <div>Balance: {formatUnits(balance || 0n, 6)} USDC</div>;
}
```

## useWriteContract Hook

Write to contracts with transaction state management:

```typescript
import { useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseUnits } from "viem";
import { useState } from "react";

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

function TransferUSDC() {
  const [recipient, setRecipient] = useState("");
  const [amount, setAmount] = useState("");
  
  const { writeContract, data: hash, isPending, isError, error } = useWriteContract();

  const handleTransfer = () => {
    writeContract({
      address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      abi: erc20Abi,
      functionName: "transfer",
      args: [recipient as `0x${string}`, parseUnits(amount, 6)]
    });
  };

  return (
    <div>
      <input
        type="text"
        placeholder="Recipient address"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
      />
      <input
        type="text"
        placeholder="Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
      />
      <button onClick={handleTransfer} disabled={isPending}>
        {isPending ? "Transferring..." : "Transfer USDC"}
      </button>
      {hash && <div>Transaction hash: {hash}</div>}
      {isError && <div>Error: {error?.message}</div>}
    </div>
  );
}
```

## useWaitForTransactionReceipt Hook

Wait for transaction confirmation:

```typescript
import { useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseUnits } from "viem";

function TransferWithConfirmation() {
  const { writeContract, data: hash } = useWriteContract();
  
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash
  });

  const handleTransfer = () => {
    writeContract({
      address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      abi: erc20Abi,
      functionName: "transfer",
      args: [
        "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb" as `0x${string}`,
        parseUnits("10", 6)
      ]
    });
  };

  return (
    <div>
      <button onClick={handleTransfer}>Transfer 10 USDC</button>
      {hash && (
        <div>
          <p>Transaction: {hash}</p>
          {isConfirming && <p>Waiting for confirmation...</p>}
          {isSuccess && <p>Transaction confirmed!</p>}
        </div>
      )}
    </div>
  );
}
```

## useSimulateContract Hook

Simulate contract write before execution:

```typescript
import { useSimulateContract, useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseUnits } from "viem";

function SafeTransfer() {
  const { data: simulationResult } = useSimulateContract({
    address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    abi: erc20Abi,
    functionName: "transfer",
    args: [
      "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb" as `0x${string}`,
      parseUnits("10", 6)
    ]
  });

  const { writeContract, data: hash } = useWriteContract();
  const { isSuccess } = useWaitForTransactionReceipt({ hash });

  const handleTransfer = () => {
    if (simulationResult?.request) {
      writeContract(simulationResult.request);
    }
  };

  return (
    <div>
      <button 
        onClick={handleTransfer} 
        disabled={!simulationResult?.request}
      >
        Transfer USDC
      </button>
      {!simulationResult?.request && (
        <p>Cannot execute transfer (insufficient balance or approval?)</p>
      )}
      {isSuccess && <p>Transfer successful!</p>}
    </div>
  );
}
```

## Complete Transfer Flow Example

```typescript
import { useAccount, useReadContract, useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseUnits, formatUnits } from "viem";
import { useState } from "react";

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
  }
] as const;

function CompleteTransfer() {
  const { address } = useAccount();
  const [recipient, setRecipient] = useState("");
  const [amount, setAmount] = useState("");

  const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";

  const { data: balance, refetch: refetchBalance } = useReadContract({
    address: USDC_ADDRESS,
    abi: erc20Abi,
    functionName: "balanceOf",
    args: address ? [address] : undefined
  });

  const { writeContract, data: hash, isPending, isError, error } = useWriteContract();
  
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
    onSuccess: () => {
      refetchBalance();
      setRecipient("");
      setAmount("");
    }
  });

  const handleTransfer = () => {
    if (!recipient || !amount) return;
    
    writeContract({
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "transfer",
      args: [recipient as `0x${string}`, parseUnits(amount, 6)]
    });
  };

  return (
    <div>
      <h2>Transfer USDC</h2>
      <p>Your Balance: {formatUnits(balance || 0n, 6)} USDC</p>
      
      <input
        type="text"
        placeholder="Recipient address"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
      />
      <input
        type="text"
        placeholder="Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
      />
      
      <button 
        onClick={handleTransfer} 
        disabled={isPending || isConfirming || !recipient || !amount}
      >
        {isPending && "Waiting for approval..."}
        {isConfirming && "Confirming transaction..."}
        {!isPending && !isConfirming && "Transfer"}
      </button>

      {hash && (
        <div>
          <p>Transaction: {hash}</p>
          <a 
            href={`https://basescan.org/tx/${hash}`} 
            target="_blank" 
            rel="noopener noreferrer"
          >
            View on Basescan
          </a>
        </div>
      )}
      
      {isSuccess && <p>Transfer successful!</p>}
      {isError && <p>Error: {error?.message}</p>}
    </div>
  );
}
```

## Token Approval Flow

```typescript
import { useAccount, useReadContract, useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseUnits, maxUint256 } from "viem";

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

function ApprovalFlow() {
  const { address } = useAccount();
  const USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";
  const AERODROME_ROUTER = "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43";

  const { data: allowance, refetch } = useReadContract({
    address: USDC_ADDRESS,
    abi: erc20Abi,
    functionName: "allowance",
    args: address && [address, AERODROME_ROUTER]
  });

  const { writeContract, data: hash } = useWriteContract();
  const { isSuccess } = useWaitForTransactionReceipt({
    hash,
    onSuccess: () => refetch()
  });

  const handleApprove = () => {
    writeContract({
      address: USDC_ADDRESS,
      abi: erc20Abi,
      functionName: "approve",
      args: [AERODROME_ROUTER, maxUint256] // Unlimited approval
    });
  };

  const needsApproval = (allowance || 0n) < parseUnits("1000", 6);

  return (
    <div>
      {needsApproval ? (
        <button onClick={handleApprove}>Approve USDC</button>
      ) : (
        <p>USDC approved for Aerodrome</p>
      )}
      {isSuccess && <p>Approval confirmed!</p>}
    </div>
  );
}
```

## Multiple Chain Configuration

```typescript
import { http, createConfig } from "wagmi";
import { base, mainnet } from "wagmi/chains";
import { coinbaseWallet, walletConnect } from "wagmi/connectors";

const config = createConfig({
  chains: [base, mainnet],
  connectors: [
    coinbaseWallet({
      appName: "Multi-Chain DApp"
    }),
    walletConnect({
      projectId: "YOUR_WALLETCONNECT_PROJECT_ID"
    })
  ],
  transports: {
    [base.id]: http("https://mainnet.base.org"),
    [mainnet.id]: http("https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY")
  }
});
```

## Common Patterns

Handling connection errors:

```typescript
import { useConnect } from "wagmi";
import { useEffect } from "react";

function ConnectionHandler() {
  const { error } = useConnect();

  useEffect(() => {
    if (error) {
      if (error.message.includes("User rejected")) {
        console.log("User cancelled connection");
      } else if (error.message.includes("Chain not configured")) {
        console.log("Please switch to Base network");
      }
    }
  }, [error]);

  return null;
}
```

Refetching data after transaction:

```typescript
import { useReadContract, useWriteContract, useWaitForTransactionReceipt } from "wagmi";

function RefetchPattern() {
  const { data: balance, refetch } = useReadContract({
    address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    abi: erc20Abi,
    functionName: "balanceOf",
    args: ["0xYOUR_ADDRESS"]
  });

  const { writeContract, data: hash } = useWriteContract();
  
  useWaitForTransactionReceipt({
    hash,
    onSuccess: () => {
      refetch(); // Refresh balance after transaction confirms
    }
  });

  return <div>Balance: {balance?.toString()}</div>;
}
```

## Gotchas

**Query Key Conflicts**: Wagmi uses TanStack Query internally. Multiple useReadContract calls with the same parameters share cache. This is usually beneficial but can cause unexpected updates.

**Account Changes**: When users switch accounts, all hooks automatically refetch. Handle loading states properly to prevent UI flicker.

**Transaction Confirmation**: useWaitForTransactionReceipt waits for 1 confirmation by default. For high-value transactions, consider waiting for more confirmations by checking receipt.confirmations.

**Error Handling**: User rejections are errors. Check error.message for "User rejected" to distinguish from actual failures and provide appropriate UI feedback.

**Smart Wallet Preference**: When using Coinbase Wallet with preference: "smartWalletOnly", ensure your contract interactions are compatible with ERC-4337 account abstraction.

**Refetch Intervals**: Setting aggressive refetchInterval values (under 5 seconds) can trigger RPC rate limits. Use WebSocket connections for real-time updates instead.

**Type Assertions**: Wagmi hooks return nullable values before data loads. Use optional chaining or null checks before accessing nested properties to prevent runtime errors.

**Chain Switching**: If user is on wrong chain, useReadContract and useWriteContract will fail. Implement chain detection and prompt users to switch networks.