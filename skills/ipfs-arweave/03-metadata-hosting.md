# Metadata Hosting Best Practices

## Overview

Metadata hosting is critical for blockchain applications, particularly NFTs, agent registries, and attestation systems. This lesson covers production strategies for hosting JSON metadata on IPFS and Arweave, including schema design, gateway reliability, caching, and compliance with standards like ERC-721, ERC-1155, and ERC-8004.

## ERC-8004 Agent Metadata

The ERC-8004 Identity Registry (0x8004A169FB4a3325136EB29fA0ceB6D2e539a432) stores a metadata URI for each agent. This URI typically points to IPFS or Arweave-hosted JSON.

**Metadata Schema:**
```typescript
interface ERC8004Metadata {
  name: string;
  description: string;
  image: string; // IPFS or Arweave URL to avatar/logo
  external_url?: string;
  attributes: Array<{
    trait_type: string;
    value: string | number;
  }>;
  agent_specific: {
    version: string;
    capabilities: string[];
    baseAddress: string;
    owner: string;
    reputation_score?: number;
    verification_status?: "verified" | "unverified" | "disputed";
  };
  contact?: {
    email?: string;
    telegram?: string;
    discord?: string;
    website?: string;
  };
  social?: {
    twitter?: string;
    github?: string;
  };
}

// Example metadata
const agentMetadata: ERC8004Metadata = {
  name: "TradeBot Alpha",
  description: "Autonomous trading agent specialized in DEX arbitrage on Base",
  image: "ipfs://QmX7Zv3h8KjYzK9P2nQ4mR5tN6wE8fH9sC1xD2yB3aG4jK",
  external_url: "https://tradebot.example.com",
  attributes: [
    { trait_type: "Agent Type", value: "Trading" },
    { trait_type: "Deployment Date", value: "2024-01-15" },
    { trait_type: "Network", value: "Base" }
  ],
  agent_specific: {
    version: "2.1.0",
    capabilities: [
      "dex-trading",
      "arbitrage",
      "liquidity-provision",
      "yield-optimization"
    ],
    baseAddress: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    owner: "0x1234567890123456789012345678901234567890",
    reputation_score: 875,
    verification_status: "verified"
  },
  contact: {
    email: "admin@tradebot.example.com",
    telegram: "@tradebot_support"
  },
  social: {
    twitter: "tradebotalpha",
    github: "tradebotalpha/contracts"
  }
};
```

**Uploading and Registering:**
```typescript
import { createWalletClient, createPublicClient, http } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";
import { PinataSDK } from "pinata-web3";

const IDENTITY_REGISTRY = "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432";

async function registerAgentWithMetadata(
  agentId: bigint,
  metadata: ERC8004Metadata,
  useArweave: boolean = false
): Promise<{ metadataURI: string; txHash: string }> {
  let metadataURI: string;
  
  if (useArweave) {
    // Permanent storage for critical agents
    const arweaveTxId = await uploadJSONToArweave(metadata, [
      { name: "Agent-ID", value: agentId.toString() },
      { name: "Standard", value: "ERC-8004" }
    ]);
    metadataURI = `ar://${arweaveTxId}`;
  } else {
    // IPFS for standard storage
    const pinata = new PinataSDK({ pinataJwt: process.env.PINATA_JWT });
    const upload = await pinata.upload.json(metadata, {
      metadata: {
        name: `agent-${agentId}-metadata`,
        keyvalues: {
          agentId: agentId.toString(),
          standard: "ERC-8004"
        }
      }
    });
    metadataURI = `ipfs://${upload.IpfsHash}`;
  }
  
  // Register on-chain
  const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
  const client = createWalletClient({
    account,
    chain: base,
    transport: http("https://mainnet.base.org")
  });
  
  const hash = await client.writeContract({
    address: IDENTITY_REGISTRY,
    abi: [{
      name: "updateMetadata",
      type: "function",
      stateMutability: "nonpayable",
      inputs: [
        { name: "agentId", type: "uint256" },
        { name: "metadataURI", type: "string" }
      ],
      outputs: []
    }],
    functionName: "updateMetadata",
    args: [agentId, metadataURI]
  });
  
  console.log(`Metadata registered for agent ${agentId}`);
  console.log(`URI: ${metadataURI}`);
  console.log(`Transaction: ${hash}`);
  
  return { metadataURI, txHash: hash };
}
```

## NFT Metadata Standards

**ERC-721 Metadata:**
```typescript
interface ERC721Metadata {
  name: string;
  description: string;
  image: string;
  external_url?: string;
  attributes?: Array<{
    trait_type: string;
    value: string | number;
    display_type?: "number" | "boost_percentage" | "boost_number" | "date";
  }>;
  background_color?: string; // 6-character hex without #
  animation_url?: string; // Video, audio, or interactive content
  youtube_url?: string;
}

const nftMetadata: ERC721Metadata = {
  name: "BAiSED Agent #1234",
  description: "An autonomous AI agent operating on Base network",
  image: "ipfs://QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG",
  external_url: "https://based-agents.xyz/agent/1234",
  attributes: [
    { trait_type: "Type", value: "Trading Agent" },
    { trait_type: "Rarity", value: "Legendary" },
    { trait_type: "Power", value: 95, display_type: "number" },
    { trait_type: "Birth Date", value: 1704067200, display_type: "date" }
  ],
  background_color: "0052FF"
};
```

**ERC-1155 Metadata:**
```typescript
interface ERC1155Metadata {
  name: string;
  description: string;
  image: string;
  properties?: {
    [key: string]: {
      name: string;
      description: string;
      image?: string;
    };
  };
  localization?: {
    uri: string; // Pattern: ipfs://QmHash/{locale}.json
    default: string;
    locales: string[];
  };
}
```

## Gateway Reliability Strategies

```typescript
class MetadataGatewayManager {
  private ipfsGateways = [
    "https://gateway.pinata.cloud/ipfs/",
    "https://ipfs.io/ipfs/",
    "https://cloudflare-ipfs.com/ipfs/",
    "https://w3s.link/ipfs/",
    "https://dweb.link/ipfs/"
  ];
  
  private arweaveGateways = [
    "https://arweave.net/",
    "https://arweave.dev/",
    "https://g8way.io/"
  ];
  
  async fetchMetadata(uri: string): Promise<any> {
    if (uri.startsWith("ipfs://")) {
      return this.fetchFromIPFS(uri.replace("ipfs://", ""));
    } else if (uri.startsWith("ar://")) {
      return this.fetchFromArweave(uri.replace("ar://", ""));
    } else if (uri.startsWith("http")) {
      return this.fetchFromHTTP(uri);
    } else {
      throw new Error(`Unsupported URI scheme: ${uri}`);
    }
  }
  
  private async fetchFromIPFS(cid: string): Promise<any> {
    const errors: Error[] = [];
    
    for (const gateway of this.ipfsGateways) {
      try {
        const controller = new AbortController();
        const timeout = setTimeout(() => controller.abort(), 5000);
        
        const response = await fetch(`${gateway}${cid}`, {
          signal: controller.signal
        });
        clearTimeout(timeout);
        
        if (response.ok) {
          return await response.json();
        }
      } catch (error) {
        errors.push(error as Error);
        continue;
      }
    }
    
    throw new Error(`All IPFS gateways failed: ${errors.map(e => e.message).join(", ")}`);
  }
  
  private async fetchFromArweave(txId: string): Promise<any> {
    for (const gateway of this.arweaveGateways) {
      try {
        const response = await fetch(`${gateway}${txId}`, {
          signal: AbortSignal.timeout(5000)
        });
        
        if (response.ok) {
          return await response.json();
        }
      } catch (error) {
        continue;
      }
    }
    
    throw new Error("All Arweave gateways failed");
  }
  
  private async fetchFromHTTP(url: string): Promise<any> {
    const response = await fetch(url, {
      signal: AbortSignal.timeout(5000)
    });
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    return await response.json();
  }
}
```

## Caching Strategies

```typescript
import { createClient } from "redis";

class MetadataCache {
  private redis;
  private ttl = 3600; // 1 hour default
  
  constructor(redisUrl: string) {
    this.redis = createClient({ url: redisUrl });
    this.redis.connect();
  }
  
  async getMetadata(uri: string): Promise<any | null> {
    try {
      const cached = await this.redis.get(`metadata:${uri}`);
      if (cached) {
        console.log(`Cache hit: ${uri}`);
        return JSON.parse(cached);
      }
    } catch (error) {
      console.error("Cache read error:", error);
    }
    return null;
  }
  
  async setMetadata(uri: string, metadata: any, customTTL?: number): Promise<void> {
    try {
      await this.redis.setEx(
        `metadata:${uri}`,
        customTTL || this.ttl,
        JSON.stringify(metadata)
      );
      console.log(`Cached: ${uri}`);
    } catch (error) {
      console.error("Cache write error:", error);
    }
  }
  
  async invalidate(uri: string): Promise<void> {
    await this.redis.del(`metadata:${uri}`);
    console.log(`Cache invalidated: ${uri}`);
  }
}

// Usage with gateway manager
class CachedMetadataService {
  private gateway: MetadataGatewayManager;
  private cache: MetadataCache;
  
  constructor(redisUrl: string) {
    this.gateway = new MetadataGatewayManager();
    this.cache = new MetadataCache(redisUrl);
  }
  
  async getAgentMetadata(agentId: bigint): Promise<ERC8004Metadata> {
    // Get URI from contract
    const uri = await this.getMetadataURI(agentId);
    
    // Check cache
    const cached = await this.cache.getMetadata(uri);
    if (cached) return cached;
    
    // Fetch from gateway
    const metadata = await this.gateway.fetchMetadata(uri);
    
    // Cache result
    await this.cache.setMetadata(uri, metadata);
    
    return metadata;
  }
  
  private async getMetadataURI(agentId: bigint): Promise<string> {
    const publicClient = createPublicClient({
      chain: base,
      transport: http("https://mainnet.base.org")
    });
    
    const uri = await publicClient.readContract({
      address: IDENTITY_REGISTRY,
      abi: [{
        name: "metadata",
        type: "function",
        stateMutability: "view",
        inputs: [{ name: "agentId", type: "uint256" }],
        outputs: [{ name: "", type: "string" }]
      }],
      functionName: "metadata",
      args: [agentId]
    });
    
    return uri as string;
  }
}
```

## Choosing IPFS vs Arweave

```typescript
interface StorageDecision {
  storage: "ipfs" | "arweave" | "both";
  reason: string;
}

function recommendStorage(
  contentType: string,
  permanence: "temporary" | "long-term" | "permanent",
  updateFrequency: "never" | "rare" | "frequent",
  budget: "low" | "medium" | "high"
): StorageDecision {
  // Permanent legal/compliance documents
  if (permanence === "permanent" && updateFrequency === "never") {
    return {
      storage: "arweave",
      reason: "Permanent immutable storage required"
    };
  }
  
  // High-value NFTs
  if (contentType === "nft" && budget === "high") {
    return {
      storage: "both",
      reason: "IPFS for performance, Arweave for permanence"
    };
  }
  
  // Frequently updated metadata
  if (updateFrequency === "frequent") {
    return {
      storage: "ipfs",
      reason: "Updates via new CIDs, cost-effective"
    };
  }
  
  // Agent registry metadata (ERC-8004)
  if (contentType === "agent-metadata") {
    return {
      storage: budget === "low" ? "ipfs" : "both",
      reason: budget === "low" 
        ? "IPFS provides sufficient reliability for agent metadata"
        : "IPFS primary with Arweave backup for critical agents"
    };
  }
  
  // Default to IPFS
  return {
    storage: "ipfs",
    reason: "General purpose, cost-effective, sufficient for most use cases"
  };
}
```

## Metadata Validation

```typescript
import Ajv from "ajv";

const erc8004Schema = {
  type: "object",
  required: ["name", "description", "image", "agent_specific"],
  properties: {
    name: { type: "string", minLength: 1, maxLength: 100 },
    description: { type: "string", minLength: 1, maxLength: 1000 },
    image: { type: "string", pattern: "^(ipfs://|ar://|https://)" },
    external_url: { type: "string", format: "uri" },
    attributes: {
      type: "array",
      items: {
        type: "object",
        required: ["trait_type", "value"],
        properties: {
          trait_type: { type: "string" },
          value: { type: ["string", "number"] }
        }
      }
    },
    agent_specific: {
      type: "object",
      required: ["version", "capabilities", "baseAddress", "owner"],
      properties: {
        version: { type: "string" },
        capabilities: {
          type: "array",
          items: { type: "string" }
        },
        baseAddress: { type: "string", pattern: "^0x[a-fA-F0-9]{40}$" },
        owner: { type: "string", pattern: "^0x[a-fA-F0-9]{40}$" },
        reputation_score: { type: "number", minimum: 0, maximum: 1000 },
        verification_status: {
          type: "string",
          enum: ["verified", "unverified", "disputed"]
        }
      }
    }
  }
};

function validateMetadata(metadata: any): { valid: boolean; errors?: string[] } {
  const ajv = new Ajv();
  const validate = ajv.compile(erc8004Schema);
  const valid = validate(metadata);
  
  if (!valid) {
    return {
      valid: false,
      errors: validate.errors?.map(err => `${err.instancePath} ${err.message}`) || []
    };
  }
  
  return { valid: true };
}
```

## Production Deployment Pattern

```typescript
class ProductionMetadataService {
  private pinata: PinataSDK;
  private irys: WebIrys | null = null;
  private cache: MetadataCache;
  private gateway: MetadataGatewayManager;
  
  constructor() {
    this.pinata = new PinataSDK({ pinataJwt: process.env.PINATA_JWT });
    this.cache = new MetadataCache(process.env.REDIS_URL!);
    this.gateway = new MetadataGatewayManager();
  }
  
  async deployAgentMetadata(
    agentId: bigint,
    metadata: ERC8004Metadata,
    options: {
      permanentBackup: boolean;
      validate: boolean;
    } = { permanentBackup: false, validate: true }
  ): Promise<{ primary: string; backup?: string; txHash: string }> {
    // Validate
    if (options.validate) {
      const validation = validateMetadata(metadata);
      if (!validation.valid) {
        throw new Error(`Invalid metadata: ${validation.errors?.join(", ")}`);
      }
    }
    
    // Upload to IPFS (primary)
    const ipfsUpload = await this.pinata.upload.json(metadata, {
      metadata: {
        name: `agent-${agentId}-metadata`,
        keyvalues: {
          agentId: agentId.toString(),
          timestamp: Date.now().toString()
        }
      },
      pinataOptions: { cidVersion: 1 }
    });
    const primaryURI = `ipfs://${ipfsUpload.IpfsHash}`;
    
    console.log(`Primary storage (IPFS): ${primaryURI}`);
    
    // Optional Arweave backup
    let backupURI: string | undefined;
    if (options.permanentBackup) {
      if (!this.irys) {
        this.irys = await initializeIrys(process.env.PRIVATE_KEY!);
      }
      
      const arweaveTxId = await uploadJSONToArweave(metadata, [
        { name: "Agent-ID", value: agentId.toString() },
        { name: "IPFS-CID", value: ipfsUpload.IpfsHash },
        { name: "Backup-Type", value: "permanent" }
      ]);
      backupURI = `ar://${arweaveTxId}`;
      
      console.log(`Backup storage (Arweave): ${backupURI}`);
    }
    
    // Register on-chain
    const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
    const client = createWalletClient({
      account,
      chain: base,
      transport: http("https://mainnet.base.org")
    });
    
    const txHash = await client.writeContract({
      address: IDENTITY_REGISTRY,
      abi: [{
        name: "updateMetadata",
        type: "function",
        stateMutability: "nonpayable",
        inputs: [
          { name: "agentId", type: "uint256" },
          { name: "metadataURI", type: "string" }
        ],
        outputs: []
      }],
      functionName: "updateMetadata",
      args: [agentId, primaryURI]
    });
    
    console.log(`Registered on-chain: ${txHash}`);
    
    // Cache the metadata
    await this.cache.setMetadata(primaryURI, metadata, 7200); // 2 hour cache
    
    return {
      primary: primaryURI,
      backup: backupURI,
      txHash
    };
  }
}
```

## Common Patterns

**Pattern 1: Metadata Migration**
```typescript
async function migrateMetadata(
  agentId: bigint,
  fromIPFS: boolean = true
): Promise<string> {
  // Fetch existing metadata
  const service = new CachedMetadataService(process.env.REDIS_URL!);
  const metadata = await service.getAgentMetadata(agentId);
  
  // Upload to new storage
  const newTxId = fromIPFS
    ? await uploadJSONToArweave(metadata)
    : await uploadToIPFS(metadata);
  
  console.log(`Migrated metadata for agent ${agentId}`);
  return newTxId;
}
```

**Pattern 2: Multi-Gateway Hosting**
```typescript
async function uploadToMultipleServices(metadata: object): Promise<string[]> {
  const results = await Promise.allSettled([
    uploadToPinata(metadata),
    uploadToWeb3Storage(metadata),
    uploadToNFTStorage(metadata)
  ]);
  
  return results
    .filter(r => r.status === "fulfilled")
    .map(r => (r as PromiseFulfilledResult<string>).value);
}
```

## Gotchas

1. **URI Scheme Consistency**: Always use "ipfs://" or "ar://" prefixes, not gateway URLs, in on-chain storage.

2. **JSON Formatting**: Use consistent JSON serialization (sorted keys, no trailing whitespace) to ensure reproducible CIDs.

3. **Image Hosting**: Large images should be uploaded separately and referenced by CID/txId in metadata.

4. **Gateway Rate Limits**: Public gateways have rate limits. Use authenticated gateways or run your own for high-traffic applications.

5. **Metadata Updates**: IPFS metadata can be "updated" by uploading new content with a new CID and updating the on-chain pointer. Plan for versioning.

6. **Schema Evolution**: Include version fields in metadata to support backward compatibility as schemas evolve.

7. **Content Policy**: Arweave is permanent. Ensure metadata complies with legal requirements before upload.

8. **Gas Optimization**: Store only the CID/txId on-chain, not the full gateway URL. Let clients construct gateway URLs.