# IPFS for Decentralized Content Storage

## Overview

IPFS (InterPlanetary File System) is a content-addressed, peer-to-peer protocol for storing and sharing data. Unlike traditional HTTP, IPFS addresses content by what it is (cryptographic hash) rather than where it is (server location). This lesson covers practical IPFS usage for BAiSED agents, including pinning services, CID generation, and reliable retrieval strategies.

## Content Addressing and CIDs

IPFS uses Content Identifiers (CIDs) to uniquely identify content. A CID is a cryptographic hash of the content itself, meaning the same content always produces the same CID.

**CID Structure:**
```
ipfs://QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       Base58 encoded multihash (CIDv0)

ipfs://bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       Base32 encoded with multicodec prefix (CIDv1)
```

**CIDv0 vs CIDv1:**
- CIDv0: Shorter, starts with "Qm", widely supported
- CIDv1: Case-insensitive, explicit codec, future-proof
- Both reference the same content and are interchangeable

## Pinning Services

IPFS content needs to be "pinned" to remain available. Pinning services provide reliable infrastructure to keep your content online.

### Pinata Implementation

```typescript
import { PinataSDK } from "pinata-web3";

const pinata = new PinataSDK({
  pinataJwt: process.env.PINATA_JWT,
  pinataGateway: "example.mypinata.cloud"
});

// Upload JSON metadata
async function uploadAgentMetadata(metadata: object): Promise<string> {
  try {
    const upload = await pinata.upload.json(metadata, {
      metadata: {
        name: "agent-metadata.json",
        keyvalues: {
          type: "erc8004-metadata",
          registry: "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432"
        }
      },
      pinataOptions: {
        cidVersion: 1
      }
    });
    
    console.log("Uploaded to IPFS:", upload.IpfsHash);
    return upload.IpfsHash; // Returns CID
  } catch (error) {
    console.error("Pinata upload failed:", error);
    throw error;
  }
}

// Upload file from buffer
async function uploadFile(fileBuffer: Buffer, filename: string): Promise<string> {
  const blob = new Blob([fileBuffer]);
  const file = new File([blob], filename);
  
  const upload = await pinata.upload.file(file, {
    metadata: {
      name: filename
    }
  });
  
  return upload.IpfsHash;
}

// Pin existing CID from another node
async function pinByCID(cid: string): Promise<void> {
  await pinata.pinning.pinByCID(cid, {
    pinataMetadata: {
      name: `pinned-${cid}`
    }
  });
  console.log("Pinned existing CID:", cid);
}
```

### web3.storage Implementation

```typescript
import { create } from "@web3-storage/w3up-client";

async function setupWeb3Storage() {
  const client = await create();
  // Login with email or load from environment
  await client.login(process.env.WEB3_STORAGE_EMAIL);
  await client.setCurrentSpace(process.env.WEB3_STORAGE_SPACE);
  return client;
}

async function uploadToWeb3Storage(data: object): Promise<string> {
  const client = await setupWeb3Storage();
  
  // Convert JSON to File
  const jsonString = JSON.stringify(data, null, 2);
  const blob = new Blob([jsonString], { type: "application/json" });
  const file = new File([blob], "metadata.json");
  
  // Upload and get CID
  const cid = await client.uploadFile(file);
  console.log("Uploaded to web3.storage:", cid.toString());
  return cid.toString();
}

// Upload directory of files
async function uploadDirectory(files: File[]): Promise<string> {
  const client = await setupWeb3Storage();
  const cid = await client.uploadDirectory(files);
  return cid.toString();
}
```

## IPFS Gateway Access

Gateways provide HTTP access to IPFS content. Use multiple gateways for reliability.

```typescript
const IPFS_GATEWAYS = [
  "https://ipfs.io/ipfs/",
  "https://cloudflare-ipfs.com/ipfs/",
  "https://gateway.pinata.cloud/ipfs/",
  "https://w3s.link/ipfs/",
  "https://dweb.link/ipfs/"
];

async function fetchFromIPFS(
  cid: string,
  timeoutMs: number = 5000
): Promise<any> {
  // Try gateways in parallel, return first success
  const fetchPromises = IPFS_GATEWAYS.map(async (gateway) => {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);
    
    try {
      const response = await fetch(`${gateway}${cid}`, {
        signal: controller.signal
      });
      clearTimeout(timeout);
      
      if (!response.ok) {
        throw new Error(`Gateway returned ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      throw new Error(`Gateway ${gateway} failed: ${error}`);
    }
  });
  
  try {
    return await Promise.any(fetchPromises);
  } catch (error) {
    throw new Error("All IPFS gateways failed");
  }
}

// Verify content matches CID
import { CID } from "multiformats/cid";
import * as raw from "multiformats/codecs/raw";
import { sha256 } from "multiformats/hashes/sha2";

async function verifyCID(content: Uint8Array, expectedCID: string): Promise<boolean> {
  const hash = await sha256.digest(content);
  const cid = CID.create(1, raw.code, hash);
  return cid.toString() === expectedCID;
}
```

## Pinning Strategies for Production

```typescript
interface PinningStrategy {
  primary: string;      // Main pinning service
  backup: string[];     // Fallback services
  monitorInterval: number; // Check pin status (ms)
}

class ResilientIPFSManager {
  private strategy: PinningStrategy;
  
  constructor(strategy: PinningStrategy) {
    this.strategy = strategy;
  }
  
  async uploadWithRedundancy(data: object): Promise<string> {
    // Upload to primary
    const cid = await this.uploadToPrimary(data);
    
    // Pin to backup services
    await Promise.all(
      this.strategy.backup.map(service => 
        this.pinToBackup(service, cid)
      )
    );
    
    // Start monitoring
    this.monitorPin(cid);
    
    return cid;
  }
  
  private async uploadToPrimary(data: object): Promise<string> {
    // Implementation depends on primary service
    const pinata = new PinataSDK({ pinataJwt: process.env.PINATA_JWT });
    const result = await pinata.upload.json(data);
    return result.IpfsHash;
  }
  
  private async pinToBackup(service: string, cid: string): Promise<void> {
    console.log(`Pinning ${cid} to backup: ${service}`);
    // Implementation for each backup service
  }
  
  private monitorPin(cid: string): void {
    setInterval(async () => {
      const status = await this.checkPinStatus(cid);
      if (!status.pinned) {
        console.warn(`CID ${cid} is not pinned, re-pinning...`);
        await this.repinContent(cid);
      }
    }, this.strategy.monitorInterval);
  }
  
  private async checkPinStatus(cid: string): Promise<{pinned: boolean}> {
    // Query pinning service API
    return { pinned: true };
  }
  
  private async repinContent(cid: string): Promise<void> {
    // Re-pin if unpinned
  }
}
```

## ERC-8004 Metadata Integration

```typescript
import { createPublicClient, createWalletClient, http } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const IDENTITY_REGISTRY = "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432";

async function updateAgentMetadataIPFS(
  agentId: bigint,
  metadata: object
): Promise<string> {
  // 1. Upload metadata to IPFS
  const pinata = new PinataSDK({ pinataJwt: process.env.PINATA_JWT });
  const upload = await pinata.upload.json(metadata);
  const cid = upload.IpfsHash;
  const metadataURI = `ipfs://${cid}`;
  
  // 2. Update on-chain registry
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
  
  console.log(`Updated metadata for agent ${agentId}`);
  console.log(`Transaction: ${hash}`);
  console.log(`IPFS CID: ${cid}`);
  console.log(`Gateway URL: https://gateway.pinata.cloud/ipfs/${cid}`);
  
  return metadataURI;
}
```

## Common Patterns

**Pattern 1: Versioned Metadata**
```typescript
interface VersionedMetadata {
  version: string;
  timestamp: number;
  previousCID?: string;
  data: object;
}

async function updateVersionedMetadata(
  agentId: bigint,
  newData: object,
  previousCID?: string
): Promise<string> {
  const metadata: VersionedMetadata = {
    version: "1.0.0",
    timestamp: Date.now(),
    previousCID,
    data: newData
  };
  
  return await updateAgentMetadataIPFS(agentId, metadata);
}
```

**Pattern 2: Batch Upload**
```typescript
async function batchUploadToIPFS(items: object[]): Promise<string[]> {
  const pinata = new PinataSDK({ pinataJwt: process.env.PINATA_JWT });
  const cids: string[] = [];
  
  for (const item of items) {
    const upload = await pinata.upload.json(item);
    cids.push(upload.IpfsHash);
    // Rate limiting
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  
  return cids;
}
```

## Gotchas

1. **Gateway Reliability**: Public gateways can be slow or unavailable. Always implement fallback logic and consider using a dedicated gateway.

2. **CID Versions**: CIDv0 and CIDv1 reference the same content but have different string representations. Convert between them when needed.

3. **Pinning Persistence**: Free pinning services may unpin content after inactivity. Monitor pin status and use paid tiers for critical data.

4. **Large Files**: Files over 100MB may have issues with some gateways. Consider chunking or using Arweave for large permanent storage.

5. **Gas Costs**: Storing full CIDs on-chain is expensive. Consider storing only the hash portion or using events to emit CIDs.

6. **Content Mutability**: IPFS content is immutable by CID, but you can update pointers (like IPNS or on-chain references) to new CIDs.

7. **JSON Formatting**: Whitespace and key ordering affect CID. Use consistent JSON serialization (e.g., JSON.stringify with sorted keys).