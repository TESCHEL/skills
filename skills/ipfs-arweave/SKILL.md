---
name: ipfs-arweave
description: Decentralized storage for agent metadata, content, and permanent data using IPFS and Arweave
trigger_phrases:
  - "upload to IPFS"
  - "store permanently on Arweave"
  - "host metadata on decentralized storage"
  - "pin content to IPFS"
  - "create permanent archive"
version: 1.0.0
---

# IPFS & Arweave Decentralized Storage

## Guidelines

This skill enables BAiSED agents to store and retrieve data using decentralized storage networks. Use this skill when:

- Storing ERC-8004 agent metadata JSON that needs to be publicly accessible and immutable
- Hosting NFT metadata for tokens minted by agents
- Creating permanent archives of agent-generated content, reports, or attestations
- Storing large files that should not be stored on-chain due to gas costs
- Building censorship-resistant applications where data availability is critical
- Creating verifiable data provenance with content-addressed storage

**When to use IPFS:**
- Metadata that may need updates (through new CIDs)
- Content that benefits from CDN-like distribution
- Files that need fast retrieval through multiple gateways
- Data that will be pinned by multiple parties for redundancy
- Cost-sensitive applications (IPFS pinning is cheaper than Arweave)

**When to use Arweave:**
- Permanent, immutable records that must never be lost
- Legal documents, attestations, or compliance records
- Historical data that has archival value
- Content where one-time payment for permanent storage is acceptable
- Data that needs cryptographic proof of permanent availability

**Storage Selection Matrix:**
- Agent registry metadata: IPFS (can update by registering new CID)
- NFT metadata: IPFS with Arweave backup for high-value assets
- Attestation evidence: Arweave (permanent proof)
- Temporary working files: IPFS only
- Legal agreements: Arweave (immutable record)

**Integration with Base:**
Store CIDs and Arweave transaction IDs on-chain in:
- ERC-8004 Identity Registry metadata field (0x8004A169FB4a3325136EB29fA0ceB6D2e539a432)
- EAS attestations as reference data (0x4200000000000000000000000000000000000021)
- NFT tokenURI fields for ERC-721/1155 tokens
- Custom contract storage for application-specific data

**Key Principles:**
1. Always verify content by CID/hash after upload
2. Use multiple IPFS pinning services for critical data
3. Consider gateway reliability and fallback options
4. Budget for Arweave storage costs (pay once, store forever)
5. Implement content addressing in your smart contracts
6. Cache frequently accessed content appropriately
7. Monitor pin status and renew/migrate as needed

## Examples

**Example 1: Store Agent Metadata**
```
Human: "Upload my agent metadata to IPFS and register it"
Agent: Uploads JSON metadata to Pinata, retrieves CID ipfs://Qm..., 
calls updateMetadata on ERC-8004 registry with the new CID, 
confirms transaction and provides gateway URLs for verification.
```

**Example 2: Create Permanent Attestation Archive**
```
Human: "Archive this attestation evidence permanently"
Agent: Packages evidence files, uploads to Arweave via Irys,
receives transaction ID, stores AR tx ID in EAS attestation data,
provides permanent URL https://arweave.net/[txId].
```

**Example 3: Host NFT Metadata**
```
Human: "Prepare NFT metadata for my collection with decentralized hosting"
Agent: Creates ERC-721 metadata JSON, uploads to both IPFS and Arweave,
returns primary IPFS CID and backup Arweave URL, suggests using IPFS
for tokenURI with Arweave as permanent backup reference.
```

## Resources

- IPFS Documentation: https://docs.ipfs.tech/
- Pinata IPFS Pinning: https://docs.pinata.cloud/
- web3.storage API: https://web3.storage/docs/
- Arweave Documentation: https://docs.arweave.org/
- Irys (formerly Bundlr): https://docs.irys.xyz/
- IPFS HTTP Gateways: https://docs.ipfs.tech/concepts/ipfs-gateway/
- Arweave GraphQL Guide: https://gql-guide.vercel.app/
- ERC-721 Metadata Standard: https://eips.ethereum.org/EIPS/eip-721
- Content Addressing Spec: https://github.com/multiformats/cid

## Cross-References

- **erc-8004-registry**: Store IPFS/Arweave CIDs in agent metadata field
- **eas-attestations**: Reference storage URLs in attestation data
- **openclaw-packaging**: Package agent code for decentralized distribution
- **nft-minting**: Host NFT metadata on IPFS/Arweave for tokenURI
- **governance-proposals**: Archive proposal documents permanently