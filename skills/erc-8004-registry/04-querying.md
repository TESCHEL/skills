# Querying the Registry

## Overview

Querying the ERC-8004 registry enables agent discovery, capability matching, and network mapping. The Identity Registry provides methods to resolve domains to token IDs, read metadata, and enumerate all registered agents. Combined with the Reputation Registry, you can build sophisticated agent selection and routing systems.

Key query patterns include:
- Domain-to-token ID resolution
- Token ID-to-domain reverse lookup
- Batch metadata fetching
- Filtering agents by capabilities or protocols
- Cross-chain agent resolution
- Reputation-based ranking

The registry is an ERC-721 contract, so standard NFT enumeration methods (if implemented) can list all agents. Off-chain indexing with tools like The Graph or Ponder provides more efficient queries for production systems.

## Domain Resolution

Resolve a domain name to a token ID and owner:

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
    name: 'domainToTokenId',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'domain', type: 'string' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'ownerOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'address' }]
  },
  {
    name: 'tokenIdToDomain',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'string' }]
  },
  {
    name: 'tokenURI',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ name: '', type: 'string' }]
  }
] as const;

interface AgentIdentity {
  domain: string;
  tokenId: bigint;
  owner: string;
  metadataURI: string;
}

async function resolveDomain(domain: string): Promise<AgentIdentity> {
  const tokenId = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: registryAbi,
    functionName: 'domainToTokenId',
    args: [domain]
  });
  
  if (tokenId === 0n) {
    throw new Error(`Domain ${domain} not registered`);
  }
  
  const [owner, metadataURI] = await Promise.all([
    publicClient.readContract({
      address: IDENTITY_REGISTRY,
      abi: registryAbi,
      functionName: 'ownerOf',
      args: [tokenId]
    }),
    publicClient.readContract({
      address: IDENTITY_REGISTRY,
      abi: registryAbi,
      functionName: 'tokenURI',
      args: [tokenId]
    })
  ]);
  
  return {
    domain,
    tokenId,
    owner,
    metadataURI
  };
}

const identity = await resolveDomain('tradingbot.agent');
console.log(`Domain: ${identity.domain}`);
console.log(`Token ID: ${identity.tokenId}`);
console.log(`Owner: ${identity.owner}`);
console.log(`Metadata: ${identity.metadataURI}`);
```

## Reverse Lookup

Find the domain for a given token ID:

```typescript
async function reverseResolve(tokenId: bigint): Promise<string> {
  const domain = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: registryAbi,
    functionName: 'tokenIdToDomain',
    args: [tokenId]
  });
  
  if (!domain) {
    throw new Error(`Token ID ${tokenId} not found`);
  }
  
  return domain;
}

const domain = await reverseResolve(42n);
console.log(`Token #42 -> ${domain}`);
```

## Fetching Full Agent Profiles

Combine identity and reputation data:

```typescript
interface AgentProfile {
  identity: AgentIdentity;
  metadata: any;
  reputation: {
    averageRating: number;
    totalRatings: bigint;
    feedbackCount: bigint;
  };
}

async function fetchAgentProfile(domain: string): Promise<AgentProfile> {
  const identity = await resolveDomain(domain);
  
  let metadataURL = identity.metadataURI;
  if (metadataURL.startsWith('ipfs://')) {
    metadataURL = metadataURL.replace('ipfs://', 'https://ipfs.io/ipfs/');
  } else if (metadataURL.startsWith('ar://')) {
    metadataURL = metadataURL.replace('ar://', 'https://arweave.net/');
  }
  
  const metadataResponse = await fetch(metadataURL);
  const metadata = await metadataResponse.json();
  
  const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63';
  
  const reputationAbi = [
    {
      name: 'getAverageRating',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'agentTokenId', type: 'uint256' }],
      outputs: [{ name: '', type: 'uint256' }]
    },
    {
      name: 'getTotalRatings',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'agentTokenId', type: 'uint256' }],
      outputs: [{ name: '', type: 'uint256' }]
    },
    {
      name: 'getFeedbackCount',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'agentTokenId', type: 'uint256' }],
      outputs: [{ name: '', type: 'uint256' }]
    }
  ] as const;
  
  const [averageRating, totalRatings, feedbackCount] = await Promise.all([
    publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: reputationAbi,
      functionName: 'getAverageRating',
      args: [identity.tokenId]
    }),
    publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: reputationAbi,
      functionName: 'getTotalRatings',
      args: [identity.tokenId]
    }),
    publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: reputationAbi,
      functionName: 'getFeedbackCount',
      args: [identity.tokenId]
    })
  ]);
  
  return {
    identity,
    metadata,
    reputation: {
      averageRating: Number(averageRating) / 100,
      totalRatings,
      feedbackCount
    }
  };
}

const profile = await fetchAgentProfile('tradingbot.agent');
console.log(`Agent: ${profile.metadata.name}`);
console.log(`Type: ${profile.metadata.type}`);
console.log(`Capabilities: ${profile.metadata.capabilities.join(', ')}`);
console.log(`Rating: ${profile.reputation.averageRating.toFixed(2)} / 5.00`);
console.log(`Feedback Count: ${profile.reputation.feedbackCount}`);
```

## Filtering by Capabilities

Find agents that support specific capabilities:

```typescript
async function findAgentsByCapability(
  domains: string[],
  requiredCapabilities: string[]
): Promise<AgentProfile[]> {
  const profiles = await Promise.all(
    domains.map(domain => fetchAgentProfile(domain).catch(() => null))
  );
  
  return profiles.filter(profile => {
    if (!profile) return false;
    
    const agentCaps = profile.metadata.capabilities || [];
    return requiredCapabilities.every(cap => agentCaps.includes(cap));
  }) as AgentProfile[];
}

const allDomains = [
  'tradingbot.agent',
  'defibot.agent',
  'helper.agent'
];

const swapAgents = await findAgentsByCapability(allDomains, ['token-swaps']);
console.log('Agents with token-swaps capability:');
swapAgents.forEach(agent => {
  console.log(`  ${agent.identity.domain}: ${agent.metadata.name}`);
});
```

## X402 Protocol Discovery

Find agents that support X402 payment protocol:

```typescript
async function findX402Agents(domains: string[]): Promise<AgentProfile[]> {
  const profiles = await Promise.all(
    domains.map(domain => fetchAgentProfile(domain).catch(() => null))
  );
  
  return profiles.filter(profile => {
    return profile && profile.metadata.x402Support === true;
  }) as AgentProfile[];
}

const x402Agents = await findX402Agents(allDomains);
console.log('X402-enabled agents:');
x402Agents.forEach(agent => {
  console.log(`  ${agent.identity.domain}`);
  console.log(`    Payment Address: ${agent.metadata.x402PaymentAddress}`);
  console.log(`    Pricing Model: ${agent.metadata.x402PricingModel}`);
});
```

## Cross-Chain Agent Resolution

The ERC-8004 standard supports cross-chain resolution using the format:
`agentRegistry:chainId:registryAddress`

Example metadata for multi-chain agents:

```json
{
  "registrations": [
    {
      "chain": "base",
      "chainId": 8453,
      "registryAddress": "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432",
      "tokenId": "42",
      "domain": "tradingbot.agent"
    },
    {
      "chain": "optimism",
      "chainId": 10,
      "registryAddress": "0x1234567890abcdef1234567890abcdef12345678",
      "tokenId": "15",
      "domain": "tradingbot.agent"
    }
  ]
}
```

Resolve cross-chain:

```typescript
import { createPublicClient, http } from 'viem';
import { optimism } from 'viem/chains';

interface ChainRegistration {
  chain: string;
  chainId: number;
  registryAddress: string;
  tokenId: string;
  domain: string;
}

async function resolveCrossChain(
  domain: string,
  targetChainId: number
): Promise<AgentIdentity | null> {
  const baseProfile = await fetchAgentProfile(domain);
  
  const registration = baseProfile.metadata.registrations.find(
    (reg: ChainRegistration) => reg.chainId === targetChainId
  );
  
  if (!registration) {
    console.log(`Agent ${domain} not registered on chain ${targetChainId}`);
    return null;
  }
  
  const chain = targetChainId === 10 ? optimism : base;
  const client = createPublicClient({
    chain,
    transport: http()
  });
  
  const tokenId = BigInt(registration.tokenId);
  const owner = await client.readContract({
    address: registration.registryAddress as `0x${string}`,
    abi: registryAbi,
    functionName: 'ownerOf',
    args: [tokenId]
  });
  
  const metadataURI = await client.readContract({
    address: registration.registryAddress as `0x${string}`,
    abi: registryAbi,
    functionName: 'tokenURI',
    args: [tokenId]
  });
  
  return {
    domain: registration.domain,
    tokenId,
    owner,
    metadataURI
  };
}

const optimismIdentity = await resolveCrossChain('tradingbot.agent', 10);
if (optimismIdentity) {
  console.log(`Found on Optimism: token #${optimismIdentity.tokenId}`);
  console.log(`Owner: ${optimismIdentity.owner}`);
}
```

## Common Patterns

**Pattern 1: Agent discovery with ranking**

```typescript
interface RankedAgent {
  profile: AgentProfile;
  score: number;
}

async function discoverAndRankAgents(
  domains: string[],
  requiredCapabilities: string[],
  minRating: number
): Promise<RankedAgent[]> {
  const profiles = await Promise.all(
    domains.map(domain => fetchAgentProfile(domain).catch(() => null))
  );
  
  const filtered = profiles.filter(profile => {
    if (!profile) return false;
    if (profile.reputation.averageRating < minRating) return false;
    
    const agentCaps = profile.metadata.capabilities || [];
    return requiredCapabilities.every(cap => agentCaps.includes(cap));
  }) as AgentProfile[];
  
  const ranked = filtered.map(profile => {
    const ratingScore = profile.reputation.averageRating / 5.0;
    const feedbackScore = Math.min(Number(profile.reputation.feedbackCount) / 20, 1.0);
    const score = (ratingScore * 0.7) + (feedbackScore * 0.3);
    
    return { profile, score };
  });
  
  ranked.sort((a, b) => b.score - a.score);
  
  return ranked;
}

const ranked = await discoverAndRankAgents(
  allDomains,
  ['token-swaps', 'liquidity-provision'],
  4.0
);

console.log('Ranked agents:');
ranked.forEach((item, idx) => {
  console.log(`${idx + 1}. ${item.profile.identity.domain} (score: ${item.score.toFixed(3)})`);
});
```

**Pattern 2: Batch metadata caching**

```typescript
class AgentRegistryCache {
  private cache: Map<string, AgentProfile> = new Map();
  private ttl: number;
  
  constructor(ttlSeconds: number = 3600) {
    this.ttl = ttlSeconds * 1000;
  }
  
  async getProfile(domain: string): Promise<AgentProfile> {
    const cached = this.cache.get(domain);
    if (cached) {
      return cached;
    }
    
    const profile = await fetchAgentProfile(domain);
    this.cache.set(domain, profile);
    
    setTimeout(() => {
      this.cache.delete(domain);
    }, this.ttl);
    
    return profile;
  }
  
  async batchGetProfiles(domains: string[]): Promise<AgentProfile[]> {
    return Promise.all(domains.map(d => this.getProfile(d)));
  }
}

const cache = new AgentRegistryCache(3600);
const profiles = await cache.batchGetProfiles(allDomains);
```

**Pattern 3: Service endpoint health checking**

```typescript
async function checkAgentHealth(profile: AgentProfile): Promise<boolean> {
  const apiServices = profile.metadata.services.filter(
    (s: any) => s.type === 'api'
  );
  
  if (apiServices.length === 0) return false;
  
  try {
    const response = await fetch(apiServices[0].url, {
      method: 'GET',
      headers: { 'Accept': 'application/json' },
      signal: AbortSignal.timeout(5000)
    });
    
    return response.ok;
  } catch (error) {
    console.error(`Health check failed for ${profile.identity.domain}:`, error);
    return false;
  }
}

const healthyAgents = [];
for (const profile of profiles) {
  const healthy = await checkAgentHealth(profile);
  if (healthy) {
    healthyAgents.push(profile);
  }
}

console.log(`${healthyAgents.length} / ${profiles.length} agents are healthy`);
```

## Gotchas

- Domain lookups return 0 for non-existent domains. Always check for tokenId === 0n.
- Fetching metadata from IPFS can be slow. Use IPFS gateway with good uptime or pin content.
- The registry does not have built-in enumeration. You must track token IDs off-chain or use events.
- Agent metadata is self-reported. Always validate service endpoints and capabilities.
- Cross-chain registrations in metadata are not verified on-chain. Trust the agent operator.
- Large batch queries can exceed RPC rate limits. Implement exponential backoff and caching.
- Metadata URIs can become stale if IPFS content is unpinned. Check for 404s and handle gracefully.
- The active field in metadata is not enforced on-chain. Agents may report active: true but be offline.
- Reputation scores do not account for Sybil attacks. Consider weighting by rater reputation.
- Token ownership can change via transfer. Always check current ownerOf before trusting historical data.