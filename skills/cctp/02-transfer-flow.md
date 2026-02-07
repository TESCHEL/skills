# CCTP Transfer Flow

## Overview

This lesson provides a complete walkthrough of executing CCTP transfers from Base to other supported chains. We cover the TokenMessenger contract, the depositForBurn flow, polling Circle's attestation service, completing transfers with receiveMessage on the destination chain, and handling edge cases. All examples use viem for type-safe contract interactions.

## TokenMessenger Contract

The TokenMessenger contract is your primary interface for CCTP operations on Base.

**Base TokenMessenger**: 0x1682Ae6375C4E4A97e4B583BC394c861A46D8962

### Contract ABI (Key Functions)

```typescript
import { Address } from 'viem';

const TOKEN_MESSENGER_ABI = [
  {
    name: 'depositForBurn',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'amount', type: 'uint256' },
      { name: 'destinationDomain', type: 'uint32' },
      { name: 'mintRecipient', type: 'bytes32' },
      { name: 'burnToken', type: 'address' }
    ],
    outputs: [
      { name: 'nonce', type: 'uint64' }
    ]
  },
  {
    name: 'depositForBurnWithCaller',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'amount', type: 'uint256' },
      { name: 'destinationDomain', type: 'uint32' },
      { name: 'mintRecipient', type: 'bytes32' },
      { name: 'burnToken', type: 'address' },
      { name: 'destinationCaller', type: 'bytes32' }
    ],
    outputs: [
      { name: 'nonce', type: 'uint64' }
    ]
  },
  {
    name: 'replaceDepositForBurn',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'originalMessage', type: 'bytes' },
      { name: 'originalAttestation', type: 'bytes' },
      { name: 'newDestinationCaller', type: 'bytes32' },
      { name: 'newMintRecipient', type: 'bytes32' }
    ],
    outputs: []
  }
] as const;

const MESSAGE_TRANSMITTER_ABI = [
  {
    name: 'receiveMessage',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'message', type: 'bytes' },
      { name: 'attestation', type: 'bytes' }
    ],
    outputs: [
      { name: 'success', type: 'bool' }
    ]
  },
  {
    name: 'usedNonces',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'hashOfMessage', type: 'bytes32' }
    ],
    outputs: [
      { name: 'used', type: 'uint256' }
    ]
  }
] as const;

// Events
const MESSAGE_SENT_EVENT = {
  name: 'MessageSent',
  type: 'event',
  inputs: [
    { name: 'message', type: 'bytes', indexed: false }
  ]
} as const;
```

### Contract Addresses by Chain

```typescript
const CCTP_CONTRACTS = {
  base: {
    tokenMessenger: '0x1682Ae6375C4E4A97e4B583BC394c861A46D8962' as Address,
    messageTransmitter: '0xAD09780d193884d503182aD4588450C416D6F9D4' as Address,
    usdc: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913' as Address,
    domain: 6
  },
  ethereum: {
    tokenMessenger: '0xBd3fa81B58Ba92a82136038B25aDec7066af3155' as Address,
    messageTransmitter: '0x0a992d191DEeC32aFe36203Ad87D7d289a738F81' as Address,
    usdc: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' as Address,
    domain: 0
  },
  arbitrum: {
    tokenMessenger: '0x19330d10D9Cc8751218eaf51E8885D058642E08A' as Address,
    messageTransmitter: '0xC30362313FBBA5cf9163F0bb16a0e01f01A896ca' as Address,
    usdc: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831' as Address,
    domain: 3
  },
  optimism: {
    tokenMessenger: '0x2B4069517957735bE00ceE0fadAE88a26365528f' as Address,
    messageTransmitter: '0x4D41f22c5a0e5c74090899E5a8Fb597a8842b3e8' as Address,
    usdc: '0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85' as Address,
    domain: 2
  },
  polygon: {
    tokenMessenger: '0x9daF8c91AEFAE50b9c0E69629D3F6Ca40cA3B3FE' as Address,
    messageTransmitter: '0xF3be9355363857F3e001be68856A2f96b4C39Ba9' as Address,
    usdc: '0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359' as Address,
    domain: 7
  },
  avalanche: {
    tokenMessenger: '0x6B25532e1060CE10cc3B0A99e5683b91BFDe6982' as Address,
    messageTransmitter: '0x8186359aF5F57FbB40c6b14A588d2A59C0C29880' as Address,
    usdc: '0xB97EF9Ef8734C71904D8002F8b6Bc66Dd9c48a6E' as Address,
    domain: 1
  }
};
```

## DepositForBurn Flow

The complete flow for initiating a cross-chain transfer from Base.

### Step 1: Approve USDC Spend

```typescript
import { createWalletClient, createPublicClient, http, parseUnits } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount('0x...');

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

const USDC_ABI = [
  {
    name: 'approve',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' }
    ],
    outputs: [{ name: 'success', type: 'bool' }]
  },
  {
    name: 'allowance',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' }
    ],
    outputs: [{ name: 'remaining', type: 'uint256' }]
  }
] as const;

async function approveUSDC(
  amount: bigint,
  spender: Address
): Promise<{ hash: Address; success: boolean }> {
  const usdc = CCTP_CONTRACTS.base.usdc;
  
  // Check current allowance
  const currentAllowance = await publicClient.readContract({
    address: usdc,
    abi: USDC_ABI,
    functionName: 'allowance',
    args: [account.address, spender]
  });
  
  if (currentAllowance >= amount) {
    console.log('Sufficient allowance already exists');
    return { hash: '0x0' as Address, success: true };
  }
  
  // Approve spending
  const hash = await walletClient.writeContract({
    address: usdc,
    abi: USDC_ABI,
    functionName: 'approve',
    args: [spender, amount]
  });
  
  // Wait for confirmation
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  return {
    hash: receipt.transactionHash,
    success: receipt.status === 'success'
  };
}
```

### Step 2: Execute DepositForBurn

```typescript
import { encodePacked, keccak256, pad } from 'viem';

interface DepositForBurnParams {
  amount: bigint;
  destinationDomain: number;
  mintRecipient: Address;
  destinationCaller?: Address; // Optional: restrict who can complete on destination
}

interface DepositForBurnResult {
  transactionHash: Address;
  messageBytes: `0x${string}`;
  messageHash: `0x${string}`;
  nonce: bigint;
}

// Convert Ethereum address to bytes32 format required by CCTP
function addressToBytes32(address: Address): `0x${string}` {
  return pad(address, { size: 32 });
}

async function depositForBurn(
  params: DepositForBurnParams
): Promise<DepositForBurnResult> {
  const { amount, destinationDomain, mintRecipient, destinationCaller } = params;
  const tokenMessenger = CCTP_CONTRACTS.base.tokenMessenger;
  const usdc = CCTP_CONTRACTS.base.usdc;
  
  // First approve USDC
  console.log(`Approving ${amount} USDC for burning...`);
  await approveUSDC(amount, tokenMessenger);
  
  // Choose function based on whether we want to restrict destination caller
  let hash: Address;
  
  if (destinationCaller) {
    // Use depositForBurnWithCaller to restrict who can complete transfer
    console.log('Executing depositForBurnWithCaller...');
    hash = await walletClient.writeContract({
      address: tokenMessenger,
      abi: TOKEN_MESSENGER_ABI,
      functionName: 'depositForBurnWithCaller',
      args: [
        amount,
        destinationDomain,
        addressToBytes32(mintRecipient),
        usdc,
        addressToBytes32(destinationCaller)
      ]
    });
  } else {
    // Use standard depositForBurn (anyone can complete)
    console.log('Executing depositForBurn...');
    hash = await walletClient.writeContract({
      address: tokenMessenger,
      abi: TOKEN_MESSENGER_ABI,
      functionName: 'depositForBurn',
      args: [
        amount,
        destinationDomain,
        addressToBytes32(mintRecipient),
        usdc
      ]
    });
  }
  
  console.log(`Transaction submitted: ${hash}`);
  
  // Wait for transaction receipt
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  if (receipt.status !== 'success') {
    throw new Error(`Transaction failed: ${hash}`);
  }
  
  // Extract MessageSent event from logs
  const messageTransmitter = CCTP_CONTRACTS.base.messageTransmitter;
  const messageSentLog = receipt.logs.find(
    log => log.address.toLowerCase() === messageTransmitter.toLowerCase()
  );
  
  if (!messageSentLog || !messageSentLog.data) {
    throw new Error('MessageSent event not found in transaction logs');
  }
  
  // The message bytes are in the event data
  const messageBytes = messageSentLog.data as `0x${string}`;
  const messageHash = keccak256(messageBytes);
  
  console.log(`Message hash: ${messageHash}`);
  console.log(`Message bytes length: ${messageBytes.length}`);
  
  return {
    transactionHash: receipt.transactionHash,
    messageBytes,
    messageHash,
    nonce: BigInt(receipt.blockNumber) // Simplified nonce tracking
  };
}
```

### Step 3: Monitor and Retrieve Attestation

```typescript
interface AttestationResponse {
  status: 'pending_confirmations' | 'complete' | 'not_found';
  attestation?: string;
}

async function pollForAttestation(
  messageHash: `0x${string}`,
  maxAttempts: number = 60,
  intervalMs: number = 15000
): Promise<string> {
  console.log(`Polling for attestation of message ${messageHash}...`);
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const response = await fetch(
        `https://iris-api.circle.com/attestations/${messageHash}`
      );
      
      if (!response.ok) {
        if (response.status === 404) {
          console.log(`Attempt ${attempt}/${maxAttempts}: Attestation not yet available`);
        } else {
          console.error(`HTTP error: ${response.status}`);
        }
      } else {
        const data: AttestationResponse = await response.json();
        
        if (data.status === 'complete' && data.attestation) {
          console.log('Attestation received!');
          return data.attestation;
        }
        
        console.log(`Attempt ${attempt}/${maxAttempts}: Status = ${data.status}`);
      }
    } catch (error) {
      console.error(`Error fetching attestation:`, error);
    }
    
    if (attempt < maxAttempts) {
      await new Promise(resolve => setTimeout(resolve, intervalMs));
    }
  }
  
  throw new Error(
    `Attestation not received after ${maxAttempts} attempts (${(maxAttempts * intervalMs) / 60000} minutes)`
  );
}

// Helper function to estimate when attestation should be available
function estimateAttestationTime(): Date {
  const now = new Date();
  const estimatedMinutes = 15; // Conservative estimate
  now.setMinutes(now.getMinutes() + estimatedMinutes);
  return now;
}
```

## Completing Transfer on Destination Chain

### Step 4: ReceiveMessage on Destination

```typescript
import { createPublicClient as createDestPublicClient } from 'viem';
import { mainnet } from 'viem/chains'; // Example: completing on Ethereum

async function completeTransferOnDestination(
  messageBytes: `0x${string}`,
  attestation: string,
  destinationChainName: keyof typeof CCTP_CONTRACTS
): Promise<{ hash: Address; success: boolean }> {
  const destConfig = CCTP_CONTRACTS[destinationChainName];
  
  // Create clients for destination chain
  const destPublicClient = createDestPublicClient({
    chain: destinationChainName === 'ethereum' ? mainnet : base, // Map appropriately
    transport: http()
  });
  
  // Note: You need a wallet client on the destination chain
  // This could be the same account or a relayer service
  const destWalletClient = createWalletClient({
    account,
    chain: destinationChainName === 'ethereum' ? mainnet : base,
    transport: http()
  });
  
  console.log(`Completing transfer on ${destinationChainName}...`);
  
  // Check if message already processed
  const messageHash = keccak256(messageBytes);
  const isUsed = await destPublicClient.readContract({
    address: destConfig.messageTransmitter,
    abi: MESSAGE_TRANSMITTER_ABI,
    functionName: 'usedNonces',
    args: [messageHash]
  });
  
  if (isUsed > 0n) {
    console.log('Message already processed on destination chain');
    return { hash: '0x0' as Address, success: true };
  }
  
  // Execute receiveMessage
  const hash = await destWalletClient.writeContract({
    address: destConfig.messageTransmitter,
    abi: MESSAGE_TRANSMITTER_ABI,
    functionName: 'receiveMessage',
    args: [messageBytes, attestation as `0x${string}`]
  });
  
  console.log(`Receive transaction submitted: ${hash}`);
  
  // Wait for confirmation
  const receipt = await destPublicClient.waitForTransactionReceipt({ hash });
  
  console.log(`Transfer completed on ${destinationChainName}!`);
  
  return {
    hash: receipt.transactionHash,
    success: receipt.status === 'success'
  };
}
```

## Complete End-to-End Example

```typescript
async function transferUSDCCrossChain(
  amountUSDC: string,
  recipientAddress: Address,
  destinationChain: keyof typeof CCTP_CONTRACTS
): Promise<void> {
  try {
    const amount = parseUnits(amountUSDC, 6); // USDC has 6 decimals
    const destinationDomain = CCTP_CONTRACTS[destinationChain].domain;
    
    console.log('=== Starting CCTP Transfer ===');
    console.log(`Amount: ${amountUSDC} USDC`);
    console.log(`From: Base`);
    console.log(`To: ${destinationChain}`);
    console.log(`Recipient: ${recipientAddress}`);
    console.log('');
    
    // Step 1: Burn on Base
    console.log('Step 1: Burning USDC on Base...');
    const burnResult = await depositForBurn({
      amount,
      destinationDomain,
      mintRecipient: recipientAddress
    });
    
    console.log(`Burn transaction: ${burnResult.transactionHash}`);
    console.log(`Message hash: ${burnResult.messageHash}`);
    console.log('');
    
    // Step 2: Wait for attestation
    console.log('Step 2: Waiting for Circle attestation...');
    console.log(`Estimated completion: ${estimateAttestationTime().toLocaleTimeString()}`);
    
    const attestation = await pollForAttestation(burnResult.messageHash);
    console.log('Attestation received!');
    console.log('');
    
    // Step 3: Mint on destination
    console.log(`Step 3: Minting USDC on ${destinationChain}...`);
    const mintResult = await completeTransferOnDestination(
      burnResult.messageBytes,
      attestation,
      destinationChain
    );
    
    console.log(`Mint transaction: ${mintResult.hash}`);
    console.log('');
    console.log('=== Transfer Complete ===');
    console.log(`${amountUSDC} USDC successfully transferred to ${recipientAddress} on ${destinationChain}`);
    
  } catch (error) {
    console.error('Transfer failed:', error);
    throw error;
  }
}

// Usage example
async function main() {
  await transferUSDCCrossChain(
    '100.00', // 100 USDC
    '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb1',
    'ethereum'
  );
}
```

## Error Handling and Edge Cases

```typescript
// Handle insufficient balance
async function checkSufficientBalance(
  address: Address,
  requiredAmount: bigint
): Promise<boolean> {
  const balance = await publicClient.readContract({
    address: CCTP_CONTRACTS.base.usdc,
    abi: USDC_ABI,
    functionName: 'balanceOf',
    args: [address]
  });
  
  if (balance < requiredAmount) {
    console.error(`Insufficient balance: ${balance} < ${requiredAmount}`);
    return false;
  }
  
  return true;
}

// Handle transaction revert scenarios
function parseRevertReason(error: any): string {
  if (error.message.includes('insufficient allowance')) {
    return 'USDC allowance not set correctly';
  }
  if (error.message.includes('burn amount exceeds balance')) {
    return 'Insufficient USDC balance';
  }
  if (error.message.includes('invalid destination domain')) {
    return 'Destination chain not supported by CCTP';
  }
  return error.message;
}

// Retry logic for network issues
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  delayMs: number = 2000
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      console.log(`Attempt ${attempt} failed, retrying in ${delayMs}ms...`);
      await new Promise(resolve => setTimeout(resolve, delayMs));
    }
  }
  throw new Error('Should not reach here');
}
```