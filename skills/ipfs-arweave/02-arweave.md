# Arweave for Permanent Data Storage

## Overview

Arweave is a decentralized storage network that enables permanent data storage with a one-time payment. Unlike IPFS which requires continuous pinning, Arweave's economic model (endowment) ensures data remains available forever. This lesson covers Arweave fundamentals, uploading via Irys (formerly Bundlr), pricing, and querying the permaweb.

## Arweave Fundamentals

**Key Concepts:**

1. **Permanent Storage**: Pay once, store forever through an economic endowment model
2. **Transaction ID**: Each upload gets a unique transaction ID (similar to CID but not content-addressed)
3. **Blockweave**: Novel blockchain structure that requires miners to store historical data
4. **AR Token**: Native token used to pay for storage
5. **GraphQL API**: Query transactions and metadata

**When to Use Arweave:**
- Legal documents and compliance records
- Attestation evidence that must be permanently verifiable
- Historical archives and provenance records
- NFT metadata for high-value assets
- Smart contract source code and audit reports
- Governance proposals and voting records

**Cost Structure:**
```
Current costs (approximate):
- 1 MB: ~0.0001 AR (~$0.005 at $50/AR)
- 100 MB: ~0.01 AR (~$0.50)
- 1 GB: ~0.1 AR (~$5.00)
- 10 GB: ~1 AR (~$50)

Prices adjust based on network capacity and AR token price.
```

## Direct Arweave Upload

```typescript
import Arweave from "arweave";
import { JWKInterface } from "arweave/node/lib/wallet";
import fs from "fs";

const arweave = Arweave.init({
  host: "arweave.net",
  port: 443,
  protocol: "https"
});

async function uploadToArweave(
  data: string | Buffer,
  contentType: string,
  tags: { name: string; value: string }[]
): Promise<string> {
  // Load wallet (JWK format)
  const walletKey: JWKInterface = JSON.parse(
    fs.readFileSync("arweave-wallet.json", "utf-8")
  );
  
  // Create transaction
  const transaction = await arweave.createTransaction(
    { data },
    walletKey
  );
  
  // Add tags for searchability
  transaction.addTag("Content-Type", contentType);
  tags.forEach(tag => {
    transaction.addTag(tag.name, tag.value);
  });
  
  // Sign transaction
  await arweave.transactions.sign(transaction, walletKey);
  
  // Get cost estimate
  const cost = arweave.ar.winstonToAr(transaction.reward);
  console.log(`Upload cost: ${cost} AR`);
  
  // Submit transaction
  const response = await arweave.transactions.post(transaction);
  
  if (response.status === 200) {
    console.log(`Transaction ID: ${transaction.id}`);
    console.log(`URL: https://arweave.net/${transaction.id}`);
    return transaction.id;
  } else {
    throw new Error(`Upload failed: ${response.status} ${response.statusText}`);
  }
}

// Wait for confirmation
async function waitForConfirmation(txId: string): Promise<void> {
  let confirmed = false;
  let attempts = 0;
  const maxAttempts = 60; // 10 minutes with 10s intervals
  
  while (!confirmed && attempts < maxAttempts) {
    try {
      const status = await arweave.transactions.getStatus(txId);
      if (status.confirmed && status.confirmed.number_of_confirmations > 0) {
        confirmed = true;
        console.log(`Transaction confirmed: ${status.confirmed.number_of_confirmations} confirmations`);
      }
    } catch (error) {
      console.log(`Waiting for confirmation... (${attempts}/${maxAttempts})`);
    }
    
    attempts++;
    await new Promise(resolve => setTimeout(resolve, 10000));
  }
  
  if (!confirmed) {
    throw new Error("Transaction not confirmed within timeout");
  }
}
```

## Irys (Bundlr) for Instant Uploads

Irys (formerly Bundlr) provides instant, scalable uploads to Arweave by bundling transactions.

```typescript
import { WebIrys } from "@irys/sdk";
import { base } from "viem/chains";

// Initialize Irys with Base network for payment
async function initializeIrys(privateKey: string): Promise<WebIrys> {
  const irys = new WebIrys({
    url: "https://node2.irys.xyz",
    token: "base-eth", // Pay with ETH on Base
    key: privateKey,
    config: {
      providerUrl: "https://mainnet.base.org"
    }
  });
  
  await irys.ready();
  
  const address = await irys.getLoadedAddress();
  console.log(`Irys wallet address: ${address}`);
  
  return irys;
}

// Upload JSON metadata
async function uploadJSONToArweave(
  metadata: object,
  tags?: { name: string; value: string }[]
): Promise<string> {
  const irys = await initializeIrys(process.env.PRIVATE_KEY!);
  
  // Check balance
  const balance = await irys.getLoadedBalance();
  console.log(`Irys balance: ${irys.utils.fromAtomic(balance)} ETH`);
  
  // Estimate cost
  const dataString = JSON.stringify(metadata);
  const dataSize = Buffer.byteLength(dataString);
  const cost = await irys.getPrice(dataSize);
  console.log(`Upload cost: ${irys.utils.fromAtomic(cost)} ETH`);
  
  // Fund if needed
  if (balance < cost) {
    const fundAmount = cost * BigInt(2); // Fund 2x cost
    console.log(`Funding Irys: ${irys.utils.fromAtomic(fundAmount)} ETH`);
    await irys.fund(fundAmount);
  }
  
  // Prepare tags
  const uploadTags = [
    { name: "Content-Type", value: "application/json" },
    { name: "App-Name", value: "BAiSED-Agent" },
    ...(tags || [])
  ];
  
  // Upload
  const receipt = await irys.upload(dataString, { tags: uploadTags });
  
  console.log(`Uploaded to Arweave via Irys`);
  console.log(`Transaction ID: ${receipt.id}`);
  console.log(`URL: https://arweave.net/${receipt.id}`);
  
  return receipt.id;
}

// Upload file
async function uploadFileToArweave(
  fileBuffer: Buffer,
  contentType: string,
  filename: string
): Promise<string> {
  const irys = await initializeIrys(process.env.PRIVATE_KEY!);
  
  const tags = [
    { name: "Content-Type", value: contentType },
    { name: "File-Name", value: filename },
    { name: "App-Name", value: "BAiSED-Agent" }
  ];
  
  const receipt = await irys.upload(fileBuffer, { tags });
  return receipt.id;
}

// Upload directory (manifest)
async function uploadDirectoryToArweave(
  files: { name: string; content: Buffer }[]
): Promise<string> {
  const irys = await initializeIrys(process.env.PRIVATE_KEY!);
  
  // Upload individual files first
  const fileIds: { [path: string]: string } = {};
  
  for (const file of files) {
    const receipt = await irys.upload(file.content, {
      tags: [
        { name: "Content-Type", value: "application/octet-stream" },
        { name: "File-Name", value: file.name }
      ]
    });
    fileIds[file.name] = receipt.id;
  }
  
  // Create manifest
  const manifest = {
    manifest: "arweave/paths",
    version: "0.1.0",
    index: {
      path: "index.html"
    },
    paths: fileIds
  };
  
  const manifestReceipt = await irys.upload(JSON.stringify(manifest), {
    tags: [
      { name: "Content-Type", value: "application/x.arweave-manifest+json" }
    ]
  });
  
  console.log(`Manifest uploaded: ${manifestReceipt.id}`);
  return manifestReceipt.id;
}
```

## Querying Arweave with GraphQL

```typescript
interface ArweaveTransaction {
  id: string;
  tags: { name: string; value: string }[];
  owner: { address: string };
  data: { size: string };
  block?: { timestamp: number; height: number };
}

async function queryArweaveTransactions(
  tags: { name: string; values: string[] }[],
  limit: number = 10
): Promise<ArweaveTransaction[]> {
  const query = `
    query {
      transactions(
        tags: [
          ${tags.map(tag => `
          {
            name: "${tag.name}",
            values: [${tag.values.map(v => `"${v}"`).join(", ")}]
          }
          `).join(",")}
        ],
        first: ${limit},
        sort: HEIGHT_DESC
      ) {
        edges {
          node {
            id
            tags {
              name
              value
            }
            owner {
              address
            }
            data {
              size
            }
            block {
              timestamp
              height
            }
          }
        }
      }
    }
  `;
  
  const response = await fetch("https://arweave.net/graphql", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query })
  });
  
  const result = await response.json();
  return result.data.transactions.edges.map((edge: any) => edge.node);
}

// Find agent attestations on Arweave
async function findAgentAttestations(agentAddress: string): Promise<string[]> {
  const transactions = await queryArweaveTransactions(
    [
      { name: "App-Name", values: ["BAiSED-Agent"] },
      { name: "Content-Type", values: ["application/json"] },
      { name: "Agent-Address", values: [agentAddress] },
      { name: "Data-Type", values: ["attestation"] }
    ],
    50
  );
  
  return transactions.map(tx => tx.id);
}

// Retrieve transaction data
async function getArweaveData(txId: string): Promise<any> {
  const response = await fetch(`https://arweave.net/${txId}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch: ${response.status}`);
  }
  
  const contentType = response.headers.get("content-type");
  if (contentType?.includes("application/json")) {
    return await response.json();
  } else {
    return await response.arrayBuffer();
  }
}
```

## Integration with EAS Attestations

```typescript
import { EAS } from "@ethereum-attestation-service/eas-sdk";
import { createWalletClient, http } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const EAS_ADDRESS = "0x4200000000000000000000000000000000000021";

async function createAttestationWithArweaveEvidence(
  schemaUID: string,
  recipient: string,
  evidenceData: object
): Promise<{ attestationUID: string; arweaveTxId: string }> {
  // 1. Upload evidence to Arweave
  const arweaveTxId = await uploadJSONToArweave(evidenceData, [
    { name: "Data-Type", value: "attestation-evidence" },
    { name: "Recipient", value: recipient },
    { name: "Timestamp", value: Date.now().toString() }
  ]);
  
  console.log(`Evidence stored on Arweave: ${arweaveTxId}`);
  
  // 2. Create attestation with Arweave reference
  const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
  const client = createWalletClient({
    account,
    chain: base,
    transport: http("https://mainnet.base.org")
  });
  
  const eas = new EAS(EAS_ADDRESS);
  eas.connect(client as any);
  
  const attestationData = {
    arweaveEvidence: `ar://${arweaveTxId}`,
    evidenceHash: await hashArweaveData(evidenceData),
    timestamp: BigInt(Date.now())
  };
  
  const tx = await eas.attest({
    schema: schemaUID,
    data: {
      recipient,
      data: attestationData,
      expirationTime: 0n,
      revocable: true
    }
  });
  
  const attestationUID = await tx.wait();
  
  console.log(`Attestation created: ${attestationUID}`);
  console.log(`Evidence URL: https://arweave.net/${arweaveTxId}`);
  
  return { attestationUID, arweaveTxId };
}

async function hashArweaveData(data: object): Promise<string> {
  const encoder = new TextEncoder();
  const dataBuffer = encoder.encode(JSON.stringify(data));
  const hashBuffer = await crypto.subtle.digest("SHA-256", dataBuffer);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return "0x" + hashArray.map(b => b.toString(16).padStart(2, "0")).join("");
}
```

## Common Patterns

**Pattern 1: Backup IPFS to Arweave**
```typescript
async function backupIPFSToArweave(ipfsCID: string): Promise<string> {
  // Fetch from IPFS
  const response = await fetch(`https://ipfs.io/ipfs/${ipfsCID}`);
  const data = await response.arrayBuffer();
  
  // Upload to Arweave
  const irys = await initializeIrys(process.env.PRIVATE_KEY!);
  const receipt = await irys.upload(Buffer.from(data), {
    tags: [
      { name: "IPFS-CID", value: ipfsCID },
      { name: "Backup-Source", value: "IPFS" },
      { name: "Content-Type", value: response.headers.get("content-type") || "" }
    ]
  });
  
  console.log(`Backed up ${ipfsCID} to Arweave: ${receipt.id}`);
  return receipt.id;
}
```

**Pattern 2: Versioned Documents**
```typescript
interface VersionedDocument {
  version: number;
  previousTxId?: string;
  timestamp: number;
  content: any;
}

async function uploadVersionedDocument(
  content: any,
  previousTxId?: string
): Promise<string> {
  const doc: VersionedDocument = {
    version: previousTxId ? await getVersionNumber(previousTxId) + 1 : 1,
    previousTxId,
    timestamp: Date.now(),
    content
  };
  
  return await uploadJSONToArweave(doc, [
    { name: "Document-Type", value: "versioned" },
    { name: "Version", value: doc.version.toString() }
  ]);
}

async function getVersionNumber(txId: string): Promise<number> {
  const data = await getArweaveData(txId);
  return data.version || 0;
}
```

## Gotchas

1. **Confirmation Time**: Direct Arweave uploads take 10-30 minutes to confirm. Use Irys for instant finality.

2. **Transaction ID vs Content Hash**: Arweave IDs are transaction-based, not content-addressed. Same content uploaded twice gets different IDs.

3. **Storage Cost Volatility**: AR token price fluctuates. Budget accordingly or use stablecoin-based solutions like Irys.

4. **Data Availability Delay**: Newly uploaded data may not be immediately available via GraphQL queries. Allow 1-2 minutes for indexing.

5. **Tag Limits**: Maximum 2048 bytes total for all tags. Keep tags concise.

6. **Wallet Management**: Arweave uses JWK format wallets. Never expose wallet files in client-side code.

7. **Content Moderation**: Arweave is permanent and immutable. Verify content before upload -- there is no deletion.

8. **Gateway Caching**: Different gateways may cache data differently. Use cache-control tags for predictable behavior.