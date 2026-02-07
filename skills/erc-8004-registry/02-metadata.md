# Agent Metadata

## Overview

Agent metadata in ERC-8004 follows a standardized JSON schema that describes the agent's capabilities, services, protocols, and operational status. Metadata should be hosted on decentralized storage (IPFS or Arweave) and referenced via the tokenURI field in the Identity Registry.

The metadata schema includes:
- Basic identity (name, description, type)
- Service endpoints (API URLs, WebSocket connections)
- Protocol support (X402 for payments, other standards)
- Active status and registration timestamps
- Optional links to websites, documentation, source code

Metadata is mutable -- the token owner can update the tokenURI at any time by calling setTokenURI. This allows agents to evolve their capabilities and endpoints without re-registering.

## Standard Metadata Schema

The ERC-8004 metadata follows this JSON structure:

```json
{
  "name": "Trading Assistant Agent",
  "description": "Autonomous trading agent specializing in DeFi strategies on Base",
  "type": "service",
  "image": "ipfs://QmY8Kd2vN9xR7fH3tL6mP4wE8sC5aF1bG2hD9eK3nJ4oM",
  "services": [
    {
      "type": "api",
      "url": "https://api.tradingagent.xyz/v1",
      "description": "RESTful API for trade execution and strategy queries"
    },
    {
      "type": "websocket",
      "url": "wss://ws.tradingagent.xyz",
      "description": "Real-time market data and trade notifications"
    }
  ],
  "x402Support": true,
  "x402PaymentAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "x402PricingModel": "per-request",
  "capabilities": [
    "token-swaps",
    "liquidity-provision",
    "yield-farming",
    "portfolio-rebalancing"
  ],
  "active": true,
  "registrations": [
    {
      "chain": "base",
      "chainId": 8453,
      "registryAddress": "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432",
      "tokenId": "42",
      "domain": "tradingbot.agent"
    }
  ],
  "website": "https://tradingagent.xyz",
  "documentation": "https://docs.tradingagent.xyz",
  "sourceCode": "https://github.com/tradingagent/agent",
  "license": "MIT"
}
```

## Hosting Metadata on IPFS

Use an IPFS client or service like Pinata, NFT.Storage, or Web3.Storage:

```typescript
import { create } from 'ipfs-http-client';
import { Buffer } from 'buffer';

const ipfs = create({
  host: 'ipfs.infura.io',
  port: 5001,
  protocol: 'https',
  headers: {
    authorization: 'Basic ' + Buffer.from('PROJECT_ID:PROJECT_SECRET').toString('base64')
  }
});

interface AgentMetadata {
  name: string;
  description: string;
  type: 'service' | 'autonomous' | 'assistant';
  image?: string;
  services: Array<{
    type: string;
    url: string;
    description: string;
  }>;
  x402Support: boolean;
  x402PaymentAddress?: string;
  x402PricingModel?: string;
  capabilities: string[];
  active: boolean;
  registrations: Array<{
    chain: string;
    chainId: number;
    registryAddress: string;
    tokenId: string;
    domain: string;
  }>;
  website?: string;
  documentation?: string;
  sourceCode?: string;
  license?: string;
}

async function uploadMetadataToIPFS(metadata: AgentMetadata): Promise<string> {
  const json = JSON.stringify(metadata, null, 2);
  const buffer = Buffer.from(json);
  
  const result = await ipfs.add(buffer, {
    pin: true
  });
  
  const ipfsURI = `ipfs://${result.path}`;
  console.log(`Metadata uploaded to ${ipfsURI}`);
  
  return ipfsURI;
}

const metadata: AgentMetadata = {
  name: 'DeFi Assistant',
  description: 'Multi-protocol DeFi agent for Base',
  type: 'service',
  services: [
    {
      type: 'api',
      url: 'https://api.defiassistant.xyz',
      description: 'DeFi operations API'
    }
  ],
  x402Support: true,
  x402PaymentAddress: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb',
  x402PricingModel: 'per-request',
  capabilities: ['swaps', 'lending', 'staking'],
  active: true,
  registrations: [
    {
      chain: 'base',
      chainId: 8453,
      registryAddress: '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432',
      tokenId: '123',
      domain: 'defibot.agent'
    }
  ],
  website: 'https://defiassistant.xyz',
  license: 'MIT'
};

const ipfsURI = await uploadMetadataToIPFS(metadata);
```

## Hosting Metadata on Arweave

For permanent storage, use Arweave via Bundlr or Irys:

```typescript
import Irys from '@irys/sdk';

const irys = new Irys({
  url: 'https://node2.irys.xyz',
  token: 'ethereum',
  key: 'YOUR_PRIVATE_KEY'
});

async function uploadMetadataToArweave(metadata: AgentMetadata): Promise<string> {
  const json = JSON.stringify(metadata, null, 2);
  
  const tags = [
    { name: 'Content-Type', value: 'application/json' },
    { name: 'App-Name', value: 'ERC-8004-Agent-Registry' },
    { name: 'Agent-Domain', value: metadata.registrations[0]?.domain || 'unknown' }
  ];
  
  const receipt = await irys.upload(json, { tags });
  
  const arweaveURI = `ar://${receipt.id}`;
  console.log(`Metadata uploaded to ${arweaveURI}`);
  
  return arweaveURI;
}

const arweaveURI = await uploadMetadataToArweave(metadata);
```

## Setting Token URI

After uploading metadata, update the token URI in the registry:

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432';

const account = privateKeyToAccount('0xYOUR_PRIVATE_KEY');

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

const setURIAbi = [
  {
    name: 'setTokenURI',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'tokenId', type: 'uint256' },
      { name: 'uri', type: 'string' }
    ],
    outputs: []
  },
  {
    name: 'ownerOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'address' }]
  },
  {
    name: 'tokenURI',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'string' }]
  }
] as const;

async function setTokenURI(tokenId: bigint, uri: string) {
  const owner = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: setURIAbi,
    functionName: 'ownerOf',
    args: [tokenId]
  });
  
  console.log(`Token ${tokenId} owned by ${owner}`);
  console.log(`Setting URI to ${uri}`);
  
  const { request } = await publicClient.simulateContract({
    account,
    address: IDENTITY_REGISTRY,
    abi: setURIAbi,
    functionName: 'setTokenURI',
    args: [tokenId, uri]
  });
  
  const hash = await walletClient.writeContract(request);
  
  console.log(`setTokenURI transaction: ${hash}`);
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  
  console.log(`Token URI updated in block ${receipt.blockNumber}`);
  
  return receipt;
}

const tokenId = 42n;
const ipfsURI = 'ipfs://QmX7Zd3fNqJ8xK9mH2vY4pR6tL8wE5sC1aF3bG9hD4eK2n';

await setTokenURI(tokenId, ipfsURI);
```

## Fetching and Parsing Metadata

Retrieve metadata from a token:

```typescript
async function fetchMetadata(tokenId: bigint): Promise<AgentMetadata> {
  const uri = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: setURIAbi,
    functionName: 'tokenURI',
    args: [tokenId]
  });
  
  console.log(`Token URI: ${uri}`);
  
  let metadataURL = uri;
  if (uri.startsWith('ipfs://')) {
    metadataURL = uri.replace('ipfs://', 'https://ipfs.io/ipfs/');
  } else if (uri.startsWith('ar://')) {
    metadataURL = uri.replace('ar://', 'https://arweave.net/');
  }
  
  const response = await fetch(metadataURL);
  const metadata = await response.json() as AgentMetadata;
  
  return metadata;
}

const metadata = await fetchMetadata(42n);
console.log(`Agent name: ${metadata.name}`);
console.log(`X402 support: ${metadata.x402Support}`);
console.log(`Capabilities: ${metadata.capabilities.join(', ')}`);
```

## Common Patterns

**Pattern 1: Update metadata with version tracking**

```typescript
interface VersionedMetadata extends AgentMetadata {
  version: string;
  previousVersions?: string[];
}

async function updateMetadataWithVersioning(
  tokenId: bigint,
  updates: Partial<AgentMetadata>
) {
  const currentMetadata = await fetchMetadata(tokenId) as VersionedMetadata;
  const currentURI = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: setURIAbi,
    functionName: 'tokenURI',
    args: [tokenId]
  });
  
  const newVersion = currentMetadata.version
    ? `${parseInt(currentMetadata.version.split('.')[0]) + 1}.0.0`
    : '1.0.0';
  
  const updatedMetadata: VersionedMetadata = {
    ...currentMetadata,
    ...updates,
    version: newVersion,
    previousVersions: [
      ...(currentMetadata.previousVersions || []),
      currentURI
    ]
  };
  
  const newURI = await uploadMetadataToIPFS(updatedMetadata);
  await setTokenURI(tokenId, newURI);
  
  console.log(`Updated to version ${newVersion}`);
}

await updateMetadataWithVersioning(42n, {
  capabilities: ['swaps', 'lending', 'staking', 'bridging'],
  active: true
});
```

**Pattern 2: Validate metadata schema**

```typescript
function validateMetadata(metadata: any): metadata is AgentMetadata {
  if (!metadata.name || typeof metadata.name !== 'string') return false;
  if (!metadata.description || typeof metadata.description !== 'string') return false;
  if (!['service', 'autonomous', 'assistant'].includes(metadata.type)) return false;
  if (!Array.isArray(metadata.services)) return false;
  if (typeof metadata.x402Support !== 'boolean') return false;
  if (!Array.isArray(metadata.capabilities)) return false;
  if (typeof metadata.active !== 'boolean') return false;
  if (!Array.isArray(metadata.registrations) || metadata.registrations.length === 0) return false;
  
  for (const service of metadata.services) {
    if (!service.type || !service.url) return false;
  }
  
  for (const reg of metadata.registrations) {
    if (!reg.chain || !reg.chainId || !reg.registryAddress || !reg.domain) return false;
  }
  
  return true;
}

const rawMetadata = await fetchMetadata(42n);
if (!validateMetadata(rawMetadata)) {
  throw new Error('Invalid metadata schema');
}
```

## Gotchas

- Only the token owner can call setTokenURI. If you transfer the NFT, you lose control over metadata.
- IPFS URIs are not guaranteed to be permanent. Pin your content or use a pinning service.
- Arweave provides permanent storage but costs more upfront.
- When using IPFS gateways (like ipfs.io), there may be delays in content propagation.
- The tokenURI is a string field with no length limit, but gas costs increase with longer URIs.
- There is no on-chain validation of metadata format. Always validate off-chain before trusting metadata.
- The x402PaymentAddress should be checksummed (mixed case) for proper validation.
- Metadata updates do not emit events. You must watch setTokenURI calls in the transaction logs.
- Always include at least one registration entry in metadata.registrations matching the actual on-chain registration.
- The active field is self-reported. Verify agent availability by checking service endpoints.