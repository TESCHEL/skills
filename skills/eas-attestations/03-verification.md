# Attestation Verification

## Overview

Verifying attestations is critical for establishing trust in EAS-based systems. This lesson covers reading attestations from the EAS contract, checking validity, decoding attestation data, and querying attestations efficiently using the GraphQL API. Proper verification ensures that attestations are authentic, current, and not revoked.

## Reading Attestations by UID

Every attestation has a unique identifier (UID) that can be used to retrieve its full data:

```typescript
import { createPublicClient, http, decodeAbiParameters, parseAbiParameters } from 'viem';
import { base } from 'viem/chains';

const EAS_CONTRACT_ADDRESS = '0x4200000000000000000000000000000000000021';

const easABI = [
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
  },
  {
    inputs: [{ name: 'uid', type: 'bytes32' }],
    name: 'isAttestationValid',
    outputs: [{ name: '', type: 'bool' }],
    stateMutability: 'view',
    type: 'function'
  }
];

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

async function getAttestation(attestationUID: `0x${string}`) {
  const attestation = await publicClient.readContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: easABI,
    functionName: 'getAttestation',
    args: [attestationUID]
  });

  return {
    uid: attestation.uid,
    schema: attestation.schema,
    time: attestation.time,
    expirationTime: attestation.expirationTime,
    revocationTime: attestation.revocationTime,
    refUID: attestation.refUID,
    recipient: attestation.recipient,
    attester: attestation.attester,
    revocable: attestation.revocable,
    data: attestation.data
  };
}

// Example usage
const attestationUID = '0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890';
const attestation = await getAttestation(attestationUID);

console.log('Attestation UID:', attestation.uid);
console.log('Schema:', attestation.schema);
console.log('Attester:', attestation.attester);
console.log('Recipient:', attestation.recipient);
console.log('Created at:', new Date(Number(attestation.time) * 1000));
console.log('Expires at:', attestation.expirationTime === 0n ? 'Never' : new Date(Number(attestation.expirationTime) * 1000));
console.log('Revoked:', attestation.revocationTime > 0n);
```

## Checking Attestation Validity

An attestation is valid if:
1. It has not been revoked (revocationTime is 0)
2. It has not expired (expirationTime is 0 or in the future)

```typescript
function isAttestationValid(
  attestation: {
    revocationTime: bigint;
    expirationTime: bigint;
  }
): boolean {
  const now = BigInt(Math.floor(Date.now() / 1000));
  
  // Check if revoked
  if (attestation.revocationTime > 0n) {
    return false;
  }
  
  // Check if expired
  if (attestation.expirationTime > 0n && attestation.expirationTime < now) {
    return false;
  }
  
  return true;
}

// You can also use the contract's built-in validity check
async function checkAttestationValid(attestationUID: `0x${string}`): Promise<boolean> {
  const isValid = await publicClient.readContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: easABI,
    functionName: 'isAttestationValid',
    args: [attestationUID]
  });
  
  return isValid;
}

const valid = await checkAttestationValid(attestationUID);
if (valid) {
  console.log('Attestation is valid');
} else {
  console.log('Attestation is invalid (revoked or expired)');
}
```

## Verifying Attester Authority

Check if the attester is an authorized party:

```typescript
function verifyAttester(
  attestation: { attester: `0x${string}` },
  authorizedAttesters: `0x${string}`[]
): boolean {
  return authorizedAttesters.some(
    addr => addr.toLowerCase() === attestation.attester.toLowerCase()
  );
}

const authorizedAttesters = [
  '0x742d35Cc6634C0532925a3b844Bc454e4438f44e',
  '0x1234567890abcdef1234567890abcdef12345678'
];

if (verifyAttester(attestation, authorizedAttesters)) {
  console.log('Attester is authorized');
} else {
  console.log('WARNING: Attester is not authorized');
}
```

## Decoding Attestation Data

Decode the raw bytes data according to the schema:

```typescript
function decodeAttestationData(
  data: `0x${string}`,
  schemaTypes: string
): any[] {
  try {
    const decoded = decodeAbiParameters(
      parseAbiParameters(schemaTypes),
      data
    );
    return decoded;
  } catch (error) {
    console.error('Failed to decode attestation data:', error);
    throw error;
  }
}

// Example: Decode skill endorsement
// Schema: "string skillName, uint8 level, string relationship, uint256 timestamp"
const skillSchemaTypes = 'string, uint8, string, uint256';
const decodedSkillData = decodeAttestationData(
  attestation.data,
  skillSchemaTypes
);

const [skillName, level, relationship, timestamp] = decodedSkillData;
console.log('Skill:', skillName);
console.log('Level:', level);
console.log('Relationship:', relationship);
console.log('Timestamp:', new Date(Number(timestamp) * 1000));

// Example: Decode project contribution
// Schema: "uint256 projectId, string description, uint32 hours"
const contributionSchemaTypes = 'uint256, string, uint32';
const decodedContribution = decodeAttestationData(
  attestation.data,
  contributionSchemaTypes
);

const [projectId, description, hours] = decodedContribution;
console.log('Project ID:', projectId);
console.log('Description:', description);
console.log('Hours:', hours);
```

## Complete Verification Function

Combine all checks into a comprehensive verification:

```typescript
interface AttestationVerificationResult {
  isValid: boolean;
  isRevoked: boolean;
  isExpired: boolean;
  isAttesterAuthorized: boolean;
  decodedData: any[] | null;
  error?: string;
}

async function verifyAttestation(
  attestationUID: `0x${string}`,
  schemaTypes: string,
  authorizedAttesters?: `0x${string}`[]
): Promise<AttestationVerificationResult> {
  try {
    const attestation = await getAttestation(attestationUID);
    
    const now = BigInt(Math.floor(Date.now() / 1000));
    const isRevoked = attestation.revocationTime > 0n;
    const isExpired = attestation.expirationTime > 0n && attestation.expirationTime < now;
    const isValid = !isRevoked && !isExpired;
    
    let isAttesterAuthorized = true;
    if (authorizedAttesters && authorizedAttesters.length > 0) {
      isAttesterAuthorized = verifyAttester(attestation, authorizedAttesters);
    }
    
    let decodedData = null;
    try {
      decodedData = decodeAttestationData(attestation.data, schemaTypes);
    } catch (e) {
      return {
        isValid: false,
        isRevoked,
        isExpired,
        isAttesterAuthorized,
        decodedData: null,
        error: 'Failed to decode attestation data'
      };
    }
    
    return {
      isValid: isValid && isAttesterAuthorized,
      isRevoked,
      isExpired,
      isAttesterAuthorized,
      decodedData
    };
  } catch (error) {
    return {
      isValid: false,
      isRevoked: false,
      isExpired: false,
      isAttesterAuthorized: false,
      decodedData: null,
      error: `Failed to retrieve attestation: ${error}`
    };
  }
}

// Usage
const verification = await verifyAttestation(
  attestationUID,
  'string, uint8, string, uint256',
  authorizedAttesters
);

if (verification.isValid) {
  console.log('Attestation is fully valid');
  console.log('Data:', verification.decodedData);
} else {
  if (verification.isRevoked) console.log('Attestation was revoked');
  if (verification.isExpired) console.log('Attestation has expired');
  if (!verification.isAttesterAuthorized) console.log('Attester is not authorized');
  if (verification.error) console.log('Error:', verification.error);
}
```

## GraphQL API Queries

The EAS GraphQL API provides efficient querying for attestations:

```typescript
const EAS_GRAPHQL_ENDPOINT = 'https://base.easscan.org/graphql';

interface AttestationQueryResult {
  attestations: Array<{
    id: string;
    attester: string;
    recipient: string;
    refUID: string;
    revocable: boolean;
    revocationTime: number;
    expirationTime: number;
    time: number;
    data: string;
    schema: {
      id: string;
      schema: string;
    };
  }>;
}

async function queryAttestationsByRecipient(
  recipient: string
): Promise<AttestationQueryResult> {
  const query = `
    query GetAttestations($recipient: String!) {
      attestations(where: { recipient: { equals: $recipient } }) {
        id
        attester
        recipient
        refUID
        revocable
        revocationTime
        expirationTime
        time
        data
        schema {
          id
          schema
        }
      }
    }
  `;

  const response = await fetch(EAS_GRAPHQL_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query,
      variables: { recipient }
    })
  });

  const result = await response.json();
  return result.data;
}

// Query attestations for a specific address
const attestations = await queryAttestationsByRecipient(
  '0x742d35Cc6634C0532925a3b844Bc454e4438f44e'
);

console.log(`Found ${attestations.attestations.length} attestations`);
attestations.attestations.forEach(att => {
  console.log('UID:', att.id);
  console.log('Attester:', att.attester);
  console.log('Created:', new Date(att.time * 1000));
  console.log('Schema:', att.schema.schema);
  console.log('---');
});
```

## Query Attestations by Schema

```typescript
async function queryAttestationsBySchema(
  schemaUID: string
): Promise<AttestationQueryResult> {
  const query = `
    query GetAttestationsBySchema($schemaId: String!) {
      attestations(where: { schemaId: { equals: $schemaId } }) {
        id
        attester
        recipient
        refUID
        revocable
        revocationTime
        expirationTime
        time
        data
        schema {
          id
          schema
        }
      }
    }
  `;

  const response = await fetch(EAS_GRAPHQL_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query,
      variables: { schemaId: schemaUID }
    })
  });

  const result = await response.json();
  return result.data;
}
```

## Query Attestations by Attester

```typescript
async function queryAttestationsByAttester(
  attester: string
): Promise<AttestationQueryResult> {
  const query = `
    query GetAttestationsByAttester($attester: String!) {
      attestations(
        where: { attester: { equals: $attester } }
        orderBy: { time: desc }
      ) {
        id
        attester
        recipient
        refUID
        revocable
        revocationTime
        expirationTime
        time
        data
        schema {
          id
          schema
        }
      }
    }
  `;

  const response = await fetch(EAS_GRAPHQL_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query,
      variables: { attester }
    })
  });

  const result = await response.json();
  return result.data;
}
```

## Common Patterns

**Batch Verification:**
```typescript
async function verifyMultipleAttestations(
  attestationUIDs: `0x${string}`[],
  schemaTypes: string
): Promise<Map<string, boolean>> {
  const results = new Map<string, boolean>();
  
  await Promise.all(
    attestationUIDs.map(async (uid) => {
      const verification = await verifyAttestation(uid, schemaTypes);
      results.set(uid, verification.isValid);
    })
  );
  
  return results;
}
```

**Attestation Chain Verification:**
```typescript
async function verifyAttestationChain(
  leafAttestationUID: `0x${string}`,
  maxDepth: number = 10
): Promise<boolean> {
  let currentUID = leafAttestationUID;
  let depth = 0;
  
  while (currentUID !== '0x0000000000000000000000000000000000000000000000000000000000000000' && depth < maxDepth) {
    const attestation = await getAttestation(currentUID);
    
    if (!isAttestationValid(attestation)) {
      return false;
    }
    
    currentUID = attestation.refUID;
    depth++;
  }
  
  return true;
}
```

## Gotchas

- **Zero Values**: RevocationTime and ExpirationTime of 0 have special meaning
- **Timestamp Units**: All timestamps are in seconds, not milliseconds
- **GraphQL Limits**: API may have rate limits or pagination requirements
- **Data Decoding**: Must know schema types to decode data correctly
- **Schema Mismatch**: Using wrong types for decoding causes errors
- **Case Sensitivity**: Addresses in GraphQL queries are case-sensitive
- **Indexer Lag**: GraphQL API may lag behind chain state by a few blocks
- **Reference Chains**: Deep reference chains can be expensive to verify
- **Revocation Propagation**: Revoking a parent doesn't auto-revoke children
- **Off-Chain Attestations**: GraphQL may include off-chain attestations with different validation requirements