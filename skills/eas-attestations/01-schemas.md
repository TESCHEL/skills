# EAS Schemas

## Overview

Schemas in the Ethereum Attestation Service define the structure and data types for attestations. Before creating an attestation, you must either use an existing schema or register a new one. Schemas are immutable once registered and are identified by a unique UID. This lesson covers the Schema Registry contract, schema format, registration process, and best practices.

## Schema Registry Contract

The Schema Registry on Base is deployed at:

```
0x4200000000000000000000000000000000000020
```

This contract manages all schema registrations and provides methods to:
- Register new schemas
- Query existing schemas by UID
- List schemas created by an address

The Schema Registry is a separate contract from EAS itself, ensuring schemas are universally accessible across all attestation use cases.

## Schema Format

Schemas are defined as comma-separated strings specifying field types and names:

```typescript
const schema = "uint256 projectId, string contributionType, uint32 hours, address contributor";
```

Supported types include all Solidity primitive types:
- `uint8`, `uint16`, `uint32`, `uint64`, `uint128`, `uint256`
- `int8`, `int16`, `int32`, `int64`, `int128`, `int256`
- `address`
- `bool`
- `bytes`, `bytes32` (any bytesN)
- `string`
- Arrays: `uint256[]`, `address[]`, etc.

Field names must be valid Solidity identifiers (alphanumeric and underscore).

## Registering a Schema

To register a schema, call the `register()` function on the Schema Registry:

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

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
  },
  {
    inputs: [{ name: 'uid', type: 'bytes32' }],
    name: 'getSchema',
    outputs: [
      {
        components: [
          { name: 'uid', type: 'bytes32' },
          { name: 'resolver', type: 'address' },
          { name: 'revocable', type: 'bool' },
          { name: 'schema', type: 'string' }
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

async function registerSchema(
  schemaString: string,
  resolverAddress: `0x${string}`,
  revocable: boolean
): Promise<`0x${string}`> {
  const hash = await walletClient.writeContract({
    address: SCHEMA_REGISTRY_ADDRESS,
    abi: schemaRegistryABI,
    functionName: 'register',
    args: [schemaString, resolverAddress, revocable]
  });

  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  console.log('Schema registered in tx:', receipt.transactionHash);

  // The schema UID is emitted in the Registered event
  // For this example, we'd need to parse logs to get the UID
  // In production, use the EAS SDK which handles this automatically

  return hash;
}

// Example usage
const skillSchema = "string skillName, uint8 level, uint256 timestamp";
const schemaUID = await registerSchema(
  skillSchema,
  '0x0000000000000000000000000000000000000000', // No resolver
  true // Revocable
);
```

## Resolver Contracts

Resolvers are optional smart contracts that implement custom validation logic for attestations. When a resolver address is specified during schema registration, the EAS contract will call the resolver's `attest()` and `revoke()` functions.

A resolver must implement the `ISchemaResolver` interface:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Attestation } from "@ethereum-attestation-service/eas-contracts/contracts/IEAS.sol";

contract CustomResolver {
    address public immutable eas;

    constructor(address _eas) {
        eas = _eas;
    }

    function attest(Attestation calldata attestation) external payable returns (bool) {
        require(msg.sender == eas, "Only EAS can call");
        
        // Custom validation logic
        // For example, check that the attester is whitelisted
        // or that the data meets certain criteria
        
        return true; // Return false to reject the attestation
    }

    function revoke(Attestation calldata attestation) external payable returns (bool) {
        require(msg.sender == eas, "Only EAS can call");
        
        // Custom revocation logic
        
        return true;
    }
}
```

Resolvers enable powerful use cases:
- Require payment to create attestations
- Whitelist specific attesters
- Validate data format or ranges
- Trigger side effects (mint NFTs, update state)
- Implement complex business logic

## Querying Schemas

To retrieve a schema by its UID:

```typescript
async function getSchema(schemaUID: `0x${string}`) {
  const schema = await publicClient.readContract({
    address: SCHEMA_REGISTRY_ADDRESS,
    abi: schemaRegistryABI,
    functionName: 'getSchema',
    args: [schemaUID]
  });

  console.log('Schema UID:', schema.uid);
  console.log('Schema string:', schema.schema);
  console.log('Resolver:', schema.resolver);
  console.log('Revocable:', schema.revocable);

  return schema;
}
```

## Common Schema Patterns

**Identity Verification Schema:**
```typescript
const identitySchema = "string verificationType, bytes32 documentHash, uint256 verifiedAt";
```

**Reputation Score Schema:**
```typescript
const reputationSchema = "uint8 score, string category, string evidence, address verifier";
```

**Event Attendance Schema:**
```typescript
const attendanceSchema = "bytes32 eventId, string eventName, uint256 attendedAt, string role";
```

**Skill Endorsement Schema:**
```typescript
const endorsementSchema = "string skillName, uint8 proficiencyLevel, string relationship, uint256 endorsedAt";
```

**Project Contribution Schema:**
```typescript
const contributionSchema = "bytes32 projectId, string contributionType, uint32 hours, string description";
```

## Schema Design Best Practices

1. **Be Specific**: Create schemas for specific use cases rather than generic ones
2. **Use Appropriate Types**: Choose the smallest type that fits your data (uint8 vs uint256)
3. **Include Timestamps**: Add timestamp fields for time-based validation
4. **Document Fields**: Maintain off-chain documentation explaining each field
5. **Consider Revocability**: Think carefully about whether attestations should be revocable
6. **Use Hashes for Large Data**: Store hashes of large documents rather than full content
7. **Plan for Versioning**: Create new schemas for major changes rather than reusing
8. **Test Encoding/Decoding**: Verify your schema works with your data before registering

## Gotchas

- **Schemas are Immutable**: Once registered, a schema cannot be modified. Plan carefully.
- **Case Sensitivity**: Field names are case-sensitive in the schema string
- **No Spaces in Types**: Use `uint256` not `uint 256`
- **Resolver Gas Costs**: Resolvers add gas cost to every attestation/revocation
- **Resolver Security**: Malicious resolvers can block attestations or extract value
- **Array Limitations**: Large arrays in attestations can exceed gas limits
- **String Encoding**: Strings are UTF-8 encoded, be mindful of character sets
- **Zero Address Resolver**: Use `0x0000000000000000000000000000000000000000` for no resolver
- **UID Uniqueness**: Schema UIDs are deterministic based on schema string and creator
- **Revocable Flag**: This flag determines if attestations using this schema can be revoked