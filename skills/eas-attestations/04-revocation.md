# Attestation Revocation

## Overview

Revocation allows attesters to invalidate previously created attestations. This is essential for scenarios where attestations become outdated, incorrect, or need to be withdrawn. This lesson covers the revocation process, checking revocation status, irrevocable schemas, and best practices for managing attestation lifecycle.

## Revocable vs Irrevocable Attestations

Attestations can be made revocable or irrevocable based on two factors:

1. **Schema Revocability**: Set when registering the schema
2. **Attestation Revocability**: Set when creating each attestation

Both must be true for an attestation to be revocable:

```typescript
// When registering a schema
const revocableSchema = true; // Allow revocations
const irrevocableSchema = false; // Attestations cannot be revoked

// When creating an attestation
const attestationRequest = {
  schema: schemaUID,
  data: {
    recipient: recipientAddress,
    expirationTime: 0n,
    revocable: true, // This specific attestation can be revoked
    refUID: '0x0000000000000000000000000000000000000000000000000000000000000000',
    data: encodedData,
    value: 0n
  }
};
```

Use cases for irrevocable attestations:
- Historical records that should never change
- Certificates that are permanently earned
- Legal agreements that cannot be withdrawn
- Audit trails requiring immutability

Use cases for revocable attestations:
- Temporary permissions or credentials
- Skills that may become outdated
- Statements that may need correction
- Memberships that can be terminated

## Revoking an Attestation

Only the original attester can revoke an attestation:

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const EAS_CONTRACT_ADDRESS = '0x4200000000000000000000000000000000000021';

const revokeABI = [
  {
    inputs: [
      {
        components: [
          { name: 'schema', type: 'bytes32' },
          {
            components: [
              { name: 'uid', type: 'bytes32' },
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
    name: 'revoke',
    outputs: [],
    stateMutability: 'payable',
    type: 'function'
  },
  {
    inputs: [
      {
        components: [
          { name: 'schema', type: 'bytes32' },
          {
            components: [
              { name: 'uid', type: 'bytes32' },
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
    name: 'multiRevoke',
    outputs: [],
    stateMutability: 'payable',
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

async function revokeAttestation(
  schemaUID: `0x${string}`,
  attestationUID: `0x${string}`
): Promise<`0x${string}`> {
  const revocationRequest = {
    schema: schemaUID,
    data: {
      uid: attestationUID,
      value: 0n // ETH to send to resolver
    }
  };

  const hash = await walletClient.writeContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: revokeABI,
    functionName: 'revoke',
    args: [revocationRequest]
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log('Attestation revoked in tx:', receipt.transactionHash);

  return hash;
}

// Example usage
const schemaUID = '0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef';
const attestationUID = '0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890';

await revokeAttestation(schemaUID, attestationUID);
```

## Checking Revocation Status

Check if an attestation has been revoked:

```typescript
const getAttestationABI = [
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

async function isAttestationRevoked(
  attestationUID: `0x${string}`
): Promise<boolean> {
  const attestation = await publicClient.readContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: getAttestationABI,
    functionName: 'getAttestation',
    args: [attestationUID]
  });

  // revocationTime > 0 means the attestation was revoked
  return attestation.revocationTime > 0n;
}

async function getRevocationTime(
  attestationUID: `0x${string}`
): Promise<Date | null> {
  const attestation = await publicClient.readContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: getAttestationABI,
    functionName: 'getAttestation',
    args: [attestationUID]
  });

  if (attestation.revocationTime === 0n) {
    return null; // Not revoked
  }

  return new Date(Number(attestation.revocationTime) * 1000);
}

// Check if attestation is revoked
const revoked = await isAttestationRevoked(attestationUID);
if (revoked) {
  const revokedAt = await getRevocationTime(attestationUID);
  console.log('Attestation was revoked at:', revokedAt);
} else {
  console.log('Attestation is still active');
}
```

## Multi-Revocations

Revoke multiple attestations in a single transaction:

```typescript
async function revokeMultipleAttestations(
  revocations: Array<{
    schemaUID: `0x${string}`;
    attestationUID: `0x${string}`;
  }>
): Promise<`0x${string}`> {
  // Group by schema for multiRevoke
  const groupedBySchema = revocations.reduce((acc, rev) => {
    if (!acc[rev.schemaUID]) {
      acc[rev.schemaUID] = [];
    }
    acc[rev.schemaUID].push(rev.attestationUID);
    return acc;
  }, {} as Record<string, `0x${string}`[]>);

  const multiRequests = Object.entries(groupedBySchema).map(([schemaUID, uids]) => ({
    schema: schemaUID as `0x${string}`,
    data: uids.map(uid => ({
      uid,
      value: 0n
    }))
  }));

  const hash = await walletClient.writeContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: revokeABI,
    functionName: 'multiRevoke',
    args: [multiRequests]
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log('Multiple attestations revoked in tx:', receipt.transactionHash);

  return hash;
}

// Revoke multiple attestations at once
await revokeMultipleAttestations([
  {
    schemaUID: '0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef',
    attestationUID: '0xaaaa...'
  },
  {
    schemaUID: '0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef',
    attestationUID: '0xbbbb...'
  },
  {
    schemaUID: '0xffff567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef',
    attestationUID: '0xcccc...'
  }
]);
```

## Revocation with Resolver

If the schema has a resolver, it will be called during revocation:

```solidity
// Example resolver contract
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Attestation } from "@ethereum-attestation-service/eas-contracts/contracts/IEAS.sol";

contract RevocationControlResolver {
    address public immutable eas;
    mapping(address => bool) public authorizedRevokers;

    constructor(address _eas) {
        eas = _eas;
    }

    function addAuthorizedRevoker(address revoker) external {
        // Add access control logic
        authorizedRevokers[revoker] = true;
    }

    function attest(Attestation calldata attestation) external payable returns (bool) {
        require(msg.sender == eas, "Only EAS");
        return true;
    }

    function revoke(Attestation calldata attestation) external payable returns (bool) {
        require(msg.sender == eas, "Only EAS");
        
        // Custom revocation logic: only authorized revokers can revoke
        require(
            authorizedRevokers[attestation.attester] || 
            attestation.attester == tx.origin,
            "Not authorized to revoke"
        );
        
        return true;
    }
}
```

## Complete Revocation Management

```typescript
interface RevocationInfo {
  isRevocable: boolean;
  isRevoked: boolean;
  revokedAt: Date | null;
  canRevoke: boolean;
  reason?: string;
}

async function getRevocationInfo(
  attestationUID: `0x${string}`,
  currentUserAddress: `0x${string}`
): Promise<RevocationInfo> {
  const attestation = await publicClient.readContract({
    address: EAS_CONTRACT_ADDRESS,
    abi: getAttestationABI,
    functionName: 'getAttestation',
    args: [attestationUID]
  });

  const isRevoked = attestation.revocationTime > 0n;
  const revokedAt = isRevoked ? new Date(Number(attestation.revocationTime) * 1000) : null;
  
  // Check if current user is the attester
  const canRevoke = 
    attestation.revocable && 
    !isRevoked && 
    attestation.attester.toLowerCase() === currentUserAddress.toLowerCase();

  let reason: string | undefined;
  if (!attestation.revocable) {
    reason = 'Attestation is irrevocable';
  } else if (isRevoked) {
    reason = 'Already revoked';
  } else if (attestation.attester.toLowerCase() !== currentUserAddress.toLowerCase()) {
    reason = 'Only the attester can revoke';
  }

  return {
    isRevocable: attestation.revocable,
    isRevoked,
    revokedAt,
    canRevoke,
    reason
  };
}

// Usage
const revocationInfo = await getRevocationInfo(
  attestationUID,
  '0x742d35Cc6634C0532925a3b844Bc454e4438f44e'
);

if (revocationInfo.canRevoke) {
  console.log('You can revoke this attestation');
  // Proceed with revocation
} else {
  console.log('Cannot revoke:', revocationInfo.reason);
}
```

## Irrevocable Schemas

Create schemas where attestations can never be revoked:

```typescript
const SCHEMA_REGISTRY_ADDRESS = '0x4200000000000000000000000000000000000020';

const schemaRegistryABI = [
  {
    inputs: [
      { name: 'schema', type: 'string' },
      { name: 'resolver', type: 'address' },
      { name: 'revocable', type: 'bool' }
    ],
    name: 'register',
    outputs: [{ name: '', type: 'bytes32' }],
    stateMutability: 'nonpayable',
    type: 'function'
  }
];

async function registerIrrevocableSchema(
  schemaString: string,
  resolverAddress: `0x${string}` = '0x0000000000000000000000000000000000000000'
): Promise<`0x${string}`> {
  const hash = await walletClient.writeContract({
    address: SCHEMA_REGISTRY_ADDRESS,
    abi: schemaRegistryABI,
    functionName: 'register',
    args: [
      schemaString,
      resolverAddress,
      false // irrevocable = false
    ]
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log('Irrevocable schema registered in tx:', receipt.transactionHash);

  return hash;
}

// Register a schema for permanent records
const permanentRecordSchema = "bytes32 documentHash, uint256 timestamp, string recordType";
await registerIrrevocableSchema(permanentRecordSchema);
```

## Common Patterns

**Graceful Deprecation:**
```typescript
// Instead of revoking, create a new attestation referencing the old one
async function deprecateAttestation(
  oldAttestationUID: `0x${string}`,
  schemaUID: `0x${string}`,
  reason: string
) {
  // Create new attestation explaining deprecation
  const deprecationData = encodeAbiParameters(
    parseAbiParameters('string, bytes32'),
    [reason, oldAttestationUID]
  );
  
  // Reference the old attestation
  return await createAttestation(
    schemaUID,
    '0x0000000000000000000000000000000000000000',
    deprecationData,
    oldAttestationUID
  );
}
```

**Conditional Revocation:**
```typescript
// Only revoke if conditions are met
async function conditionalRevoke(
  attestationUID: `0x${string}`,
  schemaUID: `0x${string}`,
  condition: () => Promise<boolean>
) {
  if (await condition()) {
    await revokeAttestation(schemaUID, attestationUID);
    console.log('Attestation revoked due to condition');
  } else {
    console.log('Condition not met, attestation remains active');
  }
}
```

**Batch Revocation with Logging:**
```typescript
async function revokeWithAudit(
  attestationUID: `0x${string}`,
  schemaUID: `0x${string}`,
  reason: string
) {
  // Log the revocation reason on-chain or off-chain
  console.log(`Revoking ${attestationUID}: ${reason}`);
  
  // Perform revocation
  const hash = await revokeAttestation(schemaUID, attestationUID);
  
  // Store audit trail
  // Could create another attestation documenting the revocation
  
  return hash;
}
```

## Best Practices

1. **Document Revocation Policy**: Clearly communicate when and why attestations may be revoked
2. **Use Expiration Over Revocation**: For time-limited claims, use expiration instead
3. **Create Revocation Reasons**: Make new attestations explaining why something was revoked
4. **Audit Trail**: Log revocations for accountability
5. **Grace Periods**: Consider warning periods before revocation
6. **Bulk Operations**: Use multiRevoke for efficiency
7. **Check Before Revoking**: Verify attestation is revocable and you're the attester
8. **Consider Alternatives**: Sometimes updating via new attestation is better than revoking

## Gotchas

- **Irrevocable Attestations**: Cannot be revoked even by attester -- be certain before creating
- **Schema vs Attestation Revocability**: Both must be true for revocation to work
- **Only Attester Can Revoke**: No one else can revoke, not even the recipient
- **Revocation is Permanent**: Once revoked, cannot be "unrevoked"
- **Gas Costs**: Revocation costs gas, consider batch operations
- **Referenced Attestations**: Revoking parent doesn't automatically revoke children
- **Resolver Failures**: If resolver reverts, revocation fails
- **Timestamp Recording**: Revocation time is automatically set by contract
- **No Revocation Reason On-Chain**: Must create separate attestations to document reasons
- **GraphQL Lag**: Revocation may not immediately appear in indexed data
- **Zero Address Check**: Cannot revoke attestation with UID 0x000...000
- **Schema Mismatch**: Must provide correct schema UID when revoking

## Error Handling

```typescript
async function safeRevoke(
  attestationUID: `0x${string}`,
  schemaUID: `0x${string}`
): Promise<{ success: boolean; error?: string; hash?: `0x${string}` }> {
  try {
    // Check if attestation exists and is revocable
    const attestation = await publicClient.readContract({
      address: EAS_CONTRACT_ADDRESS,
      abi: getAttestationABI,
      functionName: 'getAttestation',
      args: [attestationUID]
    });

    if (!attestation.revocable) {
      return { success: false, error: 'Attestation is irrevocable' };
    }

    if (attestation.revocationTime > 0n) {
      return { success: false, error: 'Attestation already revoked' };
    }

    if (attestation.attester.toLowerCase() !== account.address.toLowerCase()) {
      return { success: false, error: 'You are not the attester' };
    }

    // Proceed with revocation
    const hash = await revokeAttestation(schemaUID, attestationUID);
    return { success: true, hash };
    
  } catch (error) {
    return { 
      success: false, 
      error: `Revocation failed: ${error instanceof Error ? error.message : 'Unknown error'}` 
    };
  }
}

const result = await safeRevoke(attestationUID, schemaUID);
if (result.success) {
  console.log('Revocation successful:', result.hash);
} else {
  console.error('Revocation failed:', result.error);
}
```}