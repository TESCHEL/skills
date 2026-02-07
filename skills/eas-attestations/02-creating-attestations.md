# Creating Attestations

## Overview

Attestations are the core primitive of EAS -- verifiable statements made by one address about another address or entity. This lesson covers the EAS contract on Base, the attestation creation process, on-chain vs off-chain attestations, and detailed code examples for creating various types of attestations.

## EAS Contract on Base

The main EAS contract on Base is deployed at:

```
0x4200000000000000000000000000000000000021
```

This contract provides methods to:
- Create on-chain attestations with `attest()`
- Create multiple attestations with `multiAttest()`
- Revoke attestations with `revoke()`
- Query attestations by UID with `getAttestation()`

## AttestationRequest Structure

All attestations are created using the `AttestationRequest` struct:

```solidity
struct AttestationRequest {
    bytes32 schema;        // The schema UID
    AttestationRequestData data;
}

struct AttestationRequestData {
    address recipient;     // Who the attestation is about
    uint64 expirationTime; // Unix timestamp, 0 for no expiration
    bool revocable;        // Can this specific attestation be revoked
    bytes32 refUID;        // Reference to another attestation, 0 for none
    bytes data;            // ABI-encoded attestation data
    uint256 value;         // ETH to send to resolver
}
```

## Creating a Basic Attestation

Here's a complete example of creating an attestation:

```typescript
import { createPublicClient, createWalletClient, http, encodeAbiParameters, parseAbiParameters } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const EAS_CONTRACT_ADDRESS = '0x4200000000000000000000000000000000000021';

const easABI = [
  {
    inputs: [
      {
        components: [
          { name: 'schema', type: 'bytes32' },
          {
            components: [
              { name: 'recipient', type: 'address' },
              { name: 'expirationTime', type: 'uint64' },
              { name: 'revocable', type: 'bool' },
              { name: 'refUID', type: 'bytes32' },
              { name: 'data', type: 'bytes' },
              { name: 'value', type: 'uint256' }
            ],
            name: 'data',
            type: 'tuple'
          }
        ],
        name: 'request',
        type: 'tuple'
      }
    ],
    name: 'attest',
    outputs: [{ name: '', type: 'bytes32' }],
    stateMutability: 'payable',
    type: 'function'
  },
  {
    inputs: [{ name: 'uid', type: 'bytes32' }],
    name: 'getAttestation',
    outputs: [
      {
        components: [
          { name: 'uid', type: 'bytes32' },
          { name: 'schema', type: 'bytes32' },
          { name: 'time', type: 'uint64' },
          { name: 'expirationTime', type: 'uint64' },
          { name: 'revocationTime', type: 'uint64' },
          { name: 'refUID', type: 'bytes32' },
          { name: 'recipient', type: 'address' },
          { name: 'attester', type: 'address' },
          { name: 'revocable', type: 'bool' },
          { name: 'data', type: 'bytes' }
        ],
        name: '',
        type: 'tuple'
      }
    ],
    stateMutability: 'view',
    type: 'function'
  }
];

const account = privateKeyToAccount('0xYourPrivateKey');

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

async function createAttestation(
  schemaUID: `0x${string}`,
  recipient: `0x${string}`,
  attestationData: any,
  schemaTypes: string
): Promise<`0x${string}`> {
  // Encode the attestation data according to the schema
  const encodedData = encodeAbiParameters(
    parseAbiParameters(schemaTypes),
    attestationData
  );

  const attestationRequest = {
    schema: schemaUID,
    data: {
      recipient: recipient,
      expirationTime: 0n, // No expiration
      revocable: true,
      refUID: '0x0000000000000000000000000000000000000000000000000000000000000000',
      data: encodedData,
      value: 0n // No ETH sent to resolver
    }
  };

  const hash = await walletClient.writeContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: easABI,
    functionName: 'attest',
    args: [attestationRequest]
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log('Attestation created in tx:', receipt.transactionHash);

  // Parse the attestation UID from logs
  // In production, use EAS SDK which handles this
  return hash;
}

// Example: Create a skill endorsement attestation
const skillSchemaUID = '0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef';
const recipientAddress = '0x742d35Cc6634C0532925a3b844Bc454e4438f44e';

const attestationUID = await createAttestation(
  skillSchemaUID,
  recipientAddress,
  ['Solidity Development', 8, 'Colleague', BigInt(Date.now() / 1000)],
  'string, uint8, string, uint256' // Schema: skillName, level, relationship, timestamp
);
```

## Encoding Attestation Data

Attestation data must be ABI-encoded according to the schema:

```typescript
import { encodeAbiParameters, parseAbiParameters } from 'viem';

// For schema: "uint256 projectId, string description, uint32 hours"
const encodedData = encodeAbiParameters(
  parseAbiParameters('uint256, string, uint32'),
  [12345n, 'Implemented smart contract audit', 40]
);

// For schema: "address contributor, bool verified, bytes32 contributionHash"
const encodedData2 = encodeAbiParameters(
  parseAbiParameters('address, bool, bytes32'),
  [
    '0x742d35Cc6634C0532925a3b844Bc454e4438f44e',
    true,
    '0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890'
  ]
);

// For arrays
// Schema: "address[] contributors, uint256[] amounts"
const encodedData3 = encodeAbiParameters(
  parseAbiParameters('address[], uint256[]'),
  [
    ['0x742d35Cc6634C0532925a3b844Bc454e4438f44e', '0x123...'],
    [1000n, 2000n]
  ]
);
```

## Creating Attestations with Expiration

Set an expiration timestamp for time-limited attestations:

```typescript
async function createExpiringAttestation(
  schemaUID: `0x${string}`,
  recipient: `0x${string}`,
  attestationData: any,
  schemaTypes: string,
  expirationDate: Date
): Promise<`0x${string}`> {
  const encodedData = encodeAbiParameters(
    parseAbiParameters(schemaTypes),
    attestationData
  );

  const expirationTime = BigInt(Math.floor(expirationDate.getTime() / 1000));

  const attestationRequest = {
    schema: schemaUID,
    data: {
      recipient: recipient,
      expirationTime: expirationTime,
      revocable: true,
      refUID: '0x0000000000000000000000000000000000000000000000000000000000000000',
      data: encodedData,
      value: 0n
    }
  };

  const hash = await walletClient.writeContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: easABI,
    functionName: 'attest',
    args: [attestationRequest]
  });

  return hash;
}

// Create an attestation that expires in 30 days
const expirationDate = new Date();
expirationDate.setDate(expirationDate.getDate() + 30);

await createExpiringAttestation(
  skillSchemaUID,
  recipientAddress,
  ['JavaScript', 7, 'Manager', BigInt(Date.now() / 1000)],
  'string, uint8, string, uint256',
  expirationDate
);
```

## Creating Referenced Attestations

Reference another attestation to create attestation chains:

```typescript
async function createReferencedAttestation(
  schemaUID: `0x${string}`,
  recipient: `0x${string}`,
  attestationData: any,
  schemaTypes: string,
  refUID: `0x${string}`
): Promise<`0x${string}`> {
  const encodedData = encodeAbiParameters(
    parseAbiParameters(schemaTypes),
    attestationData
  );

  const attestationRequest = {
    schema: schemaUID,
    data: {
      recipient: recipient,
      expirationTime: 0n,
      revocable: true,
      refUID: refUID, // Reference to existing attestation
      data: encodedData,
      value: 0n
    }
  };

  const hash = await walletClient.writeContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: easABI,
    functionName: 'attest',
    args: [attestationRequest]
  });

  return hash;
}

// Create an attestation referencing a previous skill attestation
const originalAttestationUID = '0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890';
await createReferencedAttestation(
  skillSchemaUID,
  recipientAddress,
  ['Advanced Solidity', 9, 'Instructor', BigInt(Date.now() / 1000)],
  'string, uint8, string, uint256',
  originalAttestationUID
);
```

## Multi-Attestations

Create multiple attestations in a single transaction:

```typescript
const multiAttestABI = [
  {
    inputs: [
      {
        components: [
          { name: 'schema', type: 'bytes32' },
          {
            components: [
              { name: 'recipient', type: 'address' },
              { name: 'expirationTime', type: 'uint64' },
              { name: 'revocable', type: 'bool' },
              { name: 'refUID', type: 'bytes32' },
              { name: 'data', type: 'bytes' },
              { name: 'value', type: 'uint256' }
            ],
            name: 'data',
            type: 'tuple[]'
          }
        ],
        name: 'multiRequests',
        type: 'tuple[]'
      }
    ],
    name: 'multiAttest',
    outputs: [{ name: '', type: 'bytes32[]' }],
    stateMutability: 'payable',
    type: 'function'
  }
];

async function createMultipleAttestations(
  requests: Array<{
    schemaUID: `0x${string}`;
    recipient: `0x${string}`;
    data: any;
    schemaTypes: string;
  }>
): Promise<`0x${string}`> {
  const multiRequests = requests.map(req => ({
    schema: req.schemaUID,
    data: [{
      recipient: req.recipient,
      expirationTime: 0n,
      revocable: true,
      refUID: '0x0000000000000000000000000000000000000000000000000000000000000000',
      data: encodeAbiParameters(
        parseAbiParameters(req.schemaTypes),
        req.data
      ),
      value: 0n
    }]
  }));

  const hash = await walletClient.writeContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: multiAttestABI,
    functionName: 'multiAttest',
    args: [multiRequests]
  });

  return hash;
}

// Create multiple skill attestations at once
await createMultipleAttestations([
  {
    schemaUID: skillSchemaUID,
    recipient: recipientAddress,
    data: ['React', 8, 'Tech Lead', BigInt(Date.now() / 1000)],
    schemaTypes: 'string, uint8, string, uint256'
  },
  {
    schemaUID: skillSchemaUID,
    recipient: recipientAddress,
    data: ['TypeScript', 9, 'Tech Lead', BigInt(Date.now() / 1000)],
    schemaTypes: 'string, uint8, string, uint256'
  }
]);
```

## On-Chain vs Off-Chain Attestations

EAS supports both on-chain and off-chain attestations:

**On-Chain Attestations:**
- Stored directly on Base blockchain
- Fully trustless and verifiable
- Higher gas costs
- Permanent and censorship-resistant
- Use `attest()` function

**Off-Chain Attestations:**
- Signed by attester but stored off-chain
- Lower cost (only signature)
- Require trust in storage provider
- Can be selectively disclosed
- Use EAS SDK to create signed attestations

For most use cases on Base, on-chain attestations are recommended due to the low gas costs and high security guarantees.

## Common Patterns

**Mutual Attestations (Peer Reviews):**
```typescript
// Both parties attest about each other
await createAttestation(
  peerReviewSchemaUID,
  peerAddress,
  ['Excellent collaboration', 5, 'Project Partner'],
  'string, uint8, string'
);
```

**Time-Stamped Evidence:**
```typescript
// Always include timestamps for auditability
const timestamp = BigInt(Math.floor(Date.now() / 1000));
await createAttestation(
  evidenceSchemaUID,
  subjectAddress,
  [documentHash, timestamp, 'KYC Verification'],
  'bytes32, uint256, string'
);
```

**Delegated Attestations:**
```typescript
// Use account abstraction to let users attest through a smart account
// The attester is the smart account, not the user's EOA
```

## Gotchas

- **Gas Costs**: Each attestation costs gas. Batch with multiAttest() when possible.
- **Revocable Flag**: Must match schema's revocability setting
- **Zero Values**: Use 0 for no expiration, zero address for no reference
- **Data Encoding**: Incorrect ABI encoding will cause transaction revert
- **Recipient Zero Address**: Valid for attestations about concepts, not people
- **Resolver Calls**: If schema has resolver, it executes during attestation
- **Value Parameter**: Only needed if resolver requires ETH payment
- **Schema Existence**: Schema must exist before creating attestations
- **Timestamp Precision**: Expiration times are in seconds, not milliseconds
- **Reference Validation**: Referenced attestations should exist and be related