# BAiSED Agent Protocol Stack Overview

## Overview

The BAiSED Agent Protocol stack combines multiple Base primitives into a cohesive system for autonomous agent commerce. This lesson covers each component and how they integrate to enable trustless, efficient agent-to-agent and human-to-agent transactions.

## Core Components

### ERC-8004 Identity Registry

Provides verifiable on-chain identities for agents with metadata and capability tracking.

```typescript
import { createPublicClient, http, createWalletClient } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432';
const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63';

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

const account = privateKeyToAccount('0x...');
const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

// Register agent identity
const identityAbi = [{
  name: 'register',
  type: 'function',
  stateMutability: 'nonpayable',
  inputs: [
    { name: 'agentAddress', type: 'address' },
    { name: 'metadataURI', type: 'string' },
    { name: 'capabilities', type: 'bytes32[]' }
  ],
  outputs: [{ name: 'agentId', type: 'uint256' }]
}];

const capabilities = [
  '0x' + Buffer.from('smart-contract-analysis').toString('hex').padEnd(64, '0'),
  '0x' + Buffer.from('data-processing').toString('hex').padEnd(64, '0')
];

const hash = await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityAbi,
  functionName: 'register',
  args: [
    '0x742d35Cc6634C0532925a3b844Bc454e4438f44e',
    'ipfs://QmAgentMetadata...',
    capabilities
  ]
});

const receipt = await publicClient.waitForTransactionReceipt({ hash });
console.log('Agent registered:', receipt.logs[0].topics[1]);
```

### HTTP 402 Payment Protocol

Enables payment-required challenges for agent services using standard HTTP status codes.

```typescript
import express from 'express';
import { verifyMessage } from 'viem';

const app = express();

// Service endpoint protected by 402
app.post('/api/analyze-contract', async (req, res) => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    // Return 402 Payment Required with payment challenge
    return res.status(402).json({
      error: 'Payment Required',
      payment: {
        method: 'ERC-3009',
        token: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC
        amount: '1000000', // 1 USDC
        recipient: '0xServiceProvider...',
        validAfter: Math.floor(Date.now() / 1000),
        validBefore: Math.floor(Date.now() / 1000) + 3600,
        nonce: generateNonce()
      }
    });
  }
  
  // Verify payment signature
  const paymentProof = JSON.parse(Buffer.from(authHeader.split(' ')[1], 'base64').toString());
  const isValid = await verifyERC3009Transfer(paymentProof);
  
  if (!isValid) {
    return res.status(402).json({ error: 'Invalid payment proof' });
  }
  
  // Execute service
  const analysis = await analyzeContract(req.body.contractAddress);
  res.json({ analysis });
});

function generateNonce(): string {
  return '0x' + Buffer.from(Date.now().toString() + Math.random().toString()).toString('hex').slice(0, 64);
}
```

### Circle Programmable Wallets

Provide secure USDC custody for agents with API-driven transaction signing.

```typescript
import { initiateUserControlledWalletsClient } from '@circle-fin/user-controlled-wallets';

const circleClient = initiateUserControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY
});

// Create wallet for agent
const response = await circleClient.createWallet({
  idempotencyKey: 'agent-wallet-' + Date.now(),
  accountType: 'SCA',
  blockchains: ['BASE'],
  metadata: [{
    name: 'agentId',
    value: 'agent-0x742d35...' 
  }]
});

const walletId = response.data.wallet.id;
const walletAddress = response.data.wallet.address;

console.log('Circle wallet created:', walletAddress);

// Execute USDC transfer
const transferResponse = await circleClient.createTransaction({
  idempotencyKey: 'transfer-' + Date.now(),
  walletId: walletId,
  blockchain: 'BASE',
  tokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
  destinationAddress: '0xRecipient...',
  amounts: ['1000000'], // 1 USDC
  fee: {
    type: 'level',
    config: {
      feeLevel: 'MEDIUM'
    }
  }
});

console.log('Transfer initiated:', transferResponse.data.challengeId);
```

### CCTP Cross-Chain Bridging

Enables agents to move USDC between chains for multi-chain service delivery.

```typescript
const CCTP_MESSENGER = '0x1682Ae6375C4E4A97e4B583BC394c861A46D8962';
const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';

const cctpAbi = [{
  name: 'depositForBurn',
  type: 'function',
  stateMutability: 'nonpayable',
  inputs: [
    { name: 'amount', type: 'uint256' },
    { name: 'destinationDomain', type: 'uint32' },
    { name: 'mintRecipient', type: 'bytes32' },
    { name: 'burnToken', type: 'address' }
  ],
  outputs: [{ name: 'nonce', type: 'uint64' }]
}];

// Bridge USDC from Base to Ethereum (domain 0)
const mintRecipient = '0x' + agentAddress.slice(2).padStart(64, '0');

const hash = await walletClient.writeContract({
  address: CCTP_MESSENGER,
  abi: cctpAbi,
  functionName: 'depositForBurn',
  args: [
    BigInt('10000000'), // 10 USDC
    0, // Ethereum domain
    mintRecipient,
    USDC_BASE
  ]
});

const receipt = await publicClient.waitForTransactionReceipt({ hash });
console.log('CCTP burn initiated:', receipt.transactionHash);
```

### EAS Attestations

Provide verifiable claims about agent capabilities, service delivery, and reputation.

```typescript
const EAS_CONTRACT = '0x4200000000000000000000000000000000000021';

const easAbi = [{
  name: 'attest',
  type: 'function',
  stateMutability: 'payable',
  inputs: [{
    name: 'request',
    type: 'tuple',
    components: [
      { name: 'schema', type: 'bytes32' },
      { name: 'data', type: 'tuple', components: [
        { name: 'recipient', type: 'address' },
        { name: 'expirationTime', type: 'uint64' },
        { name: 'revocable', type: 'bool' },
        { name: 'refUID', type: 'bytes32' },
        { name: 'data', type: 'bytes' },
        { name: 'value', type: 'uint256' }
      ]}
    ]
  }],
  outputs: [{ name: 'uid', type: 'bytes32' }]
}];

// Create service delivery attestation
const serviceSchema = '0x1234...'; // Service delivery schema UID
const attestationData = encodeAbiParameters(
  [
    { name: 'serviceId', type: 'bytes32' },
    { name: 'quality', type: 'uint8' },
    { name: 'timestamp', type: 'uint64' }
  ],
  [
    '0xabcd...',
    95,
    BigInt(Math.floor(Date.now() / 1000))
  ]
);

const hash = await walletClient.writeContract({
  address: EAS_CONTRACT,
  abi: easAbi,
  functionName: 'attest',
  args: [{
    schema: serviceSchema,
    data: {
      recipient: agentAddress,
      expirationTime: BigInt(0),
      revocable: true,
      refUID: '0x0000000000000000000000000000000000000000000000000000000000000000',
      data: attestationData,
      value: BigInt(0)
    }
  }]
});

const receipt = await publicClient.waitForTransactionReceipt({ hash });
console.log('Attestation created:', receipt.logs[0].topics[1]);
```

### Basenames Resolution

Provides human-readable names for agent discovery and UX improvement.

```typescript
const BASENAME_REGISTRY = '0xb94704422c2a1e396835a571837aa5ae53285a95';

const registryAbi = [{
  name: 'resolver',
  type: 'function',
  stateMutability: 'view',
  inputs: [{ name: 'node', type: 'bytes32' }],
  outputs: [{ name: '', type: 'address' }]
}];

// Resolve basename to address
const name = 'agent-analyzer.base.eth';
const node = namehash(name);

const resolverAddress = await publicClient.readContract({
  address: BASENAME_REGISTRY,
  abi: registryAbi,
  functionName: 'resolver',
  args: [node]
});

const resolverAbi = [{
  name: 'addr',
  type: 'function',
  stateMutability: 'view',
  inputs: [{ name: 'node', type: 'bytes32' }],
  outputs: [{ name: '', type: 'address' }]
}];

const agentAddress = await publicClient.readContract({
  address: resolverAddress,
  abi: resolverAbi,
  functionName: 'addr',
  args: [node]
});

console.log('Agent address:', agentAddress);

function namehash(name: string): string {
  let node = '0x0000000000000000000000000000000000000000000000000000000000000000';
  if (name === '') return node;
  
  const labels = name.split('.');
  for (let i = labels.length - 1; i >= 0; i--) {
    const labelHash = keccak256(toBytes(labels[i]));
    node = keccak256(concat([toBytes(node), toBytes(labelHash)]));
  }
  return node;
}
```

### ERC-4337 Smart Wallets

Enable sponsored transactions and batched operations for efficient agent interactions.

```typescript
const ENTRYPOINT = '0x0576a174D229E3cFA37253523E645A78A0C91B57';

const entrypointAbi = [{
  name: 'handleOps',
  type: 'function',
  stateMutability: 'nonpayable',
  inputs: [
    { name: 'ops', type: 'tuple[]', components: [
      { name: 'sender', type: 'address' },
      { name: 'nonce', type: 'uint256' },
      { name: 'initCode', type: 'bytes' },
      { name: 'callData', type: 'bytes' },
      { name: 'callGasLimit', type: 'uint256' },
      { name: 'verificationGasLimit', type: 'uint256' },
      { name: 'preVerificationGas', type: 'uint256' },
      { name: 'maxFeePerGas', type: 'uint256' },
      { name: 'maxPriorityFeePerGas', type: 'uint256' },
      { name: 'paymasterAndData', type: 'bytes' },
      { name: 'signature', type: 'bytes' }
    ]},
    { name: 'beneficiary', type: 'address' }
  ],
  outputs: []
}];

// Submit sponsored user operation
const userOp = {
  sender: smartWalletAddress,
  nonce: BigInt(0),
  initCode: '0x',
  callData: encodeFunctionData({
    abi: agentAbi,
    functionName: 'executeService',
    args: [serviceParams]
  }),
  callGasLimit: BigInt(100000),
  verificationGasLimit: BigInt(100000),
  preVerificationGas: BigInt(21000),
  maxFeePerGas: BigInt(1000000000),
  maxPriorityFeePerGas: BigInt(1000000000),
  paymasterAndData: paymasterAddress + '...',
  signature: '0x...'
};

const hash = await walletClient.writeContract({
  address: ENTRYPOINT,
  abi: entrypointAbi,
  functionName: 'handleOps',
  args: [[userOp], bundlerAddress]
});

console.log('User operation executed:', hash);
```

## Integration Architecture

The stack components integrate as follows:

1. **Identity Layer**: ERC-8004 + Basenames provide discoverable, verifiable agent identities
2. **Payment Layer**: HTTP 402 + ERC-3009 + Circle wallets enable gasless, efficient payments
3. **Trust Layer**: EAS attestations provide reputation and capability verification
4. **Execution Layer**: ERC-4337 smart wallets enable sponsored, batched operations
5. **Interoperability Layer**: CCTP enables cross-chain agent commerce

## Common Patterns

**Pattern: Full Agent Onboarding**
```typescript
async function onboardAgent(metadata: AgentMetadata) {
  // 1. Create smart wallet
  const wallet = await deploySmartWallet();
  
  // 2. Create Circle wallet for USDC custody
  const circleWallet = await createCircleWallet(wallet.address);
  
  // 3. Register identity
  const agentId = await registerIdentity(wallet.address, metadata);
  
  // 4. Register basename
  await registerBasename(metadata.name + '.base.eth', wallet.address);
  
  // 5. Create capability attestations
  await createCapabilityAttestations(wallet.address, metadata.capabilities);
  
  return { wallet, circleWallet, agentId };
}
```

## Gotchas

**Gas Sponsorship Limits**: Paymasters have rate limits and spending caps. Always implement fallback to user-paid transactions.

**CCTP Delays**: Cross-chain bridging takes 10-20 minutes. Build async flows with status polling.

**Attestation Revocation**: Always check revocation status before trusting attestations. Schemas can allow revocation by attester or third parties.

**Circle Wallet Security**: API keys for Circle wallets provide full custody. Use secure key management and consider multi-party computation for production.

**HTTP 402 Not Universal**: Not all HTTP clients handle 402 status properly. Provide fallback mechanisms for legacy systems.