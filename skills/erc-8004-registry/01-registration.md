# Agent Registration

## Overview

Registering an agent on the ERC-8004 Identity Registry creates an on-chain identity represented as an ERC-721 NFT. Each agent has a unique domain name (like "assistant.agent") that maps to a token ID and owner address. Registration requires 0.005 ETH and emits an AgentRegistered event.

The registration process involves:
1. Checking domain availability
2. Preparing the registration transaction with 0.005 ETH
3. Calling newAgent(domain, agentAddress)
4. Receiving the minted ERC-721 token
5. Setting metadata via tokenURI

The Identity Registry contract is at 0x8004A169FB4a3325136EB29fA0ceB6D2e539a432 on Base (chain ID 8453). The agent address becomes the NFT owner and controls all future updates to metadata and ownership transfers.

## Contract Interface

The Identity Registry implements ERC-721 with additional agent-specific methods:

```typescript
interface IAgentIdentityRegistry {
  function newAgent(string calldata domain, address agentAddress) external payable returns (uint256 tokenId);
  function domainToTokenId(string calldata domain) external view returns (uint256);
  function tokenIdToDomain(uint256 tokenId) external view returns (string memory);
  function isAvailable(string calldata domain) external view returns (bool);
  function registrationFee() external view returns (uint256);
  function ownerOf(uint256 tokenId) external view returns (address);
  function tokenURI(uint256 tokenId) external view returns (string memory);
  function setTokenURI(uint256 tokenId, string calldata uri) external;
}
```

## Checking Domain Availability

Before registering, verify the domain is not already taken:

```typescript
import { createPublicClient, http } from 'viem';
import { base } from 'viem/chains';

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432';

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

const registryAbi = [
  {
    name: 'isAvailable',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'domain', type: 'string' }],
    outputs: [{ name: '', type: 'bool' }]
  },
  {
    name: 'domainToTokenId',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'domain', type: 'string' }],
    outputs: [{ name: '', type: 'uint256' }]
  }
] as const;

async function checkDomainAvailability(domain: string): Promise<boolean> {
  const available = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: registryAbi,
    functionName: 'isAvailable',
    args: [domain]
  });
  
  return available;
}

const domain = 'tradingbot.agent';
const isAvailable = await checkDomainAvailability(domain);

if (!isAvailable) {
  const tokenId = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: registryAbi,
    functionName: 'domainToTokenId',
    args: [domain]
  });
  console.log(`Domain ${domain} is taken by token ID ${tokenId}`);
}
```

## Registering a New Agent

To register an agent, send 0.005 ETH with the newAgent call:

```typescript
import { createWalletClient, http, parseEther } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

const account = privateKeyToAccount('0xYOUR_PRIVATE_KEY');

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

const registrationAbi = [
  {
    name: 'newAgent',
    type: 'function',
    stateMutability: 'payable',
    inputs: [
      { name: 'domain', type: 'string' },
      { name: 'agentAddress', type: 'address' }
    ],
    outputs: [{ name: 'tokenId', type: 'uint256' }]
  },
  {
    name: 'registrationFee',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  }
] as const;

async function registerAgent(domain: string, agentAddress: string) {
  const fee = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: registrationAbi,
    functionName: 'registrationFee'
  });
  
  console.log(`Registration fee: ${fee} wei (0.005 ETH)`);
  
  const { request } = await publicClient.simulateContract({
    account,
    address: IDENTITY_REGISTRY,
    abi: registrationAbi,
    functionName: 'newAgent',
    args: [domain, agentAddress],
    value: fee
  });
  
  const hash = await walletClient.writeContract(request);
  
  console.log(`Registration transaction: ${hash}`);
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  console.log(`Agent registered in block ${receipt.blockNumber}`);
  
  return receipt;
}

const domain = 'myagent.agent';
const agentAddress = '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb';

await registerAgent(domain, agentAddress);
```

## Extracting Token ID from Receipt

The newAgent function returns the token ID, which is also emitted in the Transfer event (from ERC-721):

```typescript
import { decodeEventLog } from 'viem';

const transferEventAbi = {
  name: 'Transfer',
  type: 'event',
  inputs: [
    { name: 'from', type: 'address', indexed: true },
    { name: 'to', type: 'address', indexed: true },
    { name: 'tokenId', type: 'uint256', indexed: true }
  ]
} as const;

function extractTokenId(receipt: any): bigint | null {
  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: [transferEventAbi],
        data: log.data,
        topics: log.topics
      });
      
      if (decoded.eventName === 'Transfer' && decoded.args.from === '0x0000000000000000000000000000000000000000') {
        return decoded.args.tokenId;
      }
    } catch (e) {
      continue;
    }
  }
  return null;
}

const receipt = await registerAgent('helper.agent', agentAddress);
const tokenId = extractTokenId(receipt);

console.log(`Minted token ID: ${tokenId}`);
```

## Common Patterns

**Pattern 1: Register and immediately set metadata**

```typescript
async function registerWithMetadata(domain: string, agentAddress: string, metadataURI: string) {
  const receipt = await registerAgent(domain, agentAddress);
  const tokenId = extractTokenId(receipt);
  
  if (!tokenId) throw new Error('Failed to extract token ID');
  
  const setURIAbi = [{
    name: 'setTokenURI',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'tokenId', type: 'uint256' },
      { name: 'uri', type: 'string' }
    ],
    outputs: []
  }] as const;
  
  const { request } = await publicClient.simulateContract({
    account,
    address: IDENTITY_REGISTRY,
    abi: setURIAbi,
    functionName: 'setTokenURI',
    args: [tokenId, metadataURI]
  });
  
  const hash = await walletClient.writeContract(request);
  await publicClient.waitForTransactionReceipt({ hash });
  
  console.log(`Metadata set for token ${tokenId}`);
  
  return tokenId;
}

const ipfsURI = 'ipfs://QmX7Zd3fNqJ8xK9mH2vY4pR6tL8wE5sC1aF3bG9hD4eK2n';
await registerWithMetadata('databot.agent', agentAddress, ipfsURI);
```

**Pattern 2: Batch availability check**

```typescript
async function checkMultipleDomains(domains: string[]): Promise<Record<string, boolean>> {
  const results = await Promise.all(
    domains.map(domain => checkDomainAvailability(domain))
  );
  
  return domains.reduce((acc, domain, idx) => {
    acc[domain] = results[idx];
    return acc;
  }, {} as Record<string, boolean>);
}

const candidates = ['bot1.agent', 'bot2.agent', 'bot3.agent'];
const availability = await checkMultipleDomains(candidates);

console.log('Available domains:', Object.entries(availability).filter(([_, v]) => v).map(([k]) => k));
```

## Gotchas

- The registration fee is exactly 0.005 ETH (5000000000000000 wei). Sending less will revert, sending more will not refund the excess.
- Domain names are case-sensitive in the contract. Conventionally use lowercase.
- The agentAddress parameter becomes the NFT owner. Make sure this address is controlled by the agent or operator.
- You cannot register the same domain twice. Always check availability first.
- After registration, the token has no metadata URI by default. You must call setTokenURI separately.
- Only the token owner can call setTokenURI. If you transfer the NFT, the new owner controls metadata.
- Domain names should follow DNS-like conventions (alphanumeric, hyphens, dots) but the contract does not enforce format.
- The contract does not have a burn or deregister function. Registrations are permanent.
- Gas costs vary but expect around 150,000-200,000 gas for registration plus the 0.005 ETH fee.