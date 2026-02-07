---
name: eas-attestations
description: Create, verify, and manage on-chain attestations using Ethereum Attestation Service on Base
trigger_phrases:
  - "create an attestation"
  - "verify attestation"
  - "register EAS schema"
  - "revoke attestation"
  - "check attestation status"
version: 1.0.0
---

# EAS Attestations on Base

## Guidelines

The Ethereum Attestation Service (EAS) provides a universal infrastructure for on-chain attestations on Base. This skill covers creating schemas, making attestations, verifying attestation data, and managing attestation lifecycle including revocation.

Use this skill when you need to:
- Create verifiable on-chain statements about entities, actions, or data
- Register custom attestation schemas for structured data
- Build reputation or identity systems using attestations
- Verify the authenticity and validity of existing attestations
- Revoke or update previously made attestations
- Query attestation history for addresses or schemas

Key contracts on Base:
- EAS Contract: 0x4200000000000000000000000000000000000021
- Schema Registry: 0x4200000000000000000000000000000000000020

EAS attestations are permanent on-chain records that can include:
- Structured data following registered schemas
- Attester address (who made the attestation)
- Recipient address (who the attestation is about)
- Expiration timestamps
- Revocation status
- Reference to other attestations

When creating attestations:
- Always register a schema first or use an existing schema UID
- Choose between on-chain and off-chain attestations based on permanence needs
- Consider gas costs -- on-chain attestations cost more but are more trustworthy
- Use revocable vs irrevocable based on whether you need to update/invalidate later
- Include expiration times for time-limited claims
- Reference other attestations to build attestation graphs

When verifying attestations:
- Check the attester address matches expected authority
- Verify the attestation has not been revoked
- Check expiration timestamp if present
- Validate the schema matches expected structure
- Decode the attestation data according to schema types

Best practices:
- Use specific, well-defined schemas rather than generic ones
- Document schema fields and their meanings
- Implement resolver contracts for complex validation logic
- Query the EAS GraphQL API for efficient attestation lookups
- Cache attestation UIDs for frequently accessed data
- Consider privacy implications of on-chain attestations

## Examples

**Example 1: User wants to create a skill verification attestation**

Trigger: "Create an attestation that I completed the Solidity basics course"

Action:
1. Check if a suitable schema exists or register a new one
2. Prepare AttestationRequest with schema UID, recipient, and encoded data
3. Call EAS.attest() with the request
4. Return the attestation UID and transaction hash
5. Explain how to verify the attestation later

**Example 2: User wants to verify an attestation's validity**

Trigger: "Check if attestation 0x1234...abcd is still valid and not revoked"

Action:
1. Query EAS contract to get attestation data by UID
2. Check revocationTime is 0 (not revoked)
3. Check expirationTime is 0 or greater than current timestamp
4. Display attester, recipient, schema, and decoded data
5. Confirm validity status

**Example 3: User wants to register a custom attestation schema**

Trigger: "Register a schema for project contributions with fields: projectId, contributionType, hours"

Action:
1. Format schema string with proper types (uint256, string, uint32)
2. Choose resolver address (or zero address for no resolver)
3. Call SchemaRegistry.register() with schema and revocable flag
4. Return schema UID for future attestations
5. Explain how to use the schema in attestations

## Resources

- EAS Documentation: https://docs.attest.sh/docs/welcome
- EAS SDK: https://github.com/ethereum-attestation-service/eas-sdk
- Base EAS Explorer: https://base.easscan.org/
- Schema Registry Spec: https://docs.attest.sh/docs/core--concepts/schemas
- GraphQL API: https://docs.attest.sh/docs/developer-tools/api
- EAS Contract Source: https://github.com/ethereum-attestation-service/eas-contracts

## Cross-References

- **erc-8004-registry**: ERC-8004 Identity Registry uses EAS attestations for identity claims
- **agent-protocol**: Agents can use EAS to attest to task completion or agent reputation
- **solidity**: Understanding Solidity is essential for creating custom resolver contracts
- **basenames**: Basenames can be linked to identities verified through EAS attestations
- **account-abstraction**: Smart accounts can make attestations on behalf of users