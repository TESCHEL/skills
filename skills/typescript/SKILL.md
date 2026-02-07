---
name: typescript
description: TypeScript for Web3 agent development with strict typing and async patterns
trigger_phrases:
  - "typescript types"
  - "async typescript"
  - "web3 typescript"
  - "viem types"
  - "strict mode typescript"
version: 1.0.0
---
# TypeScript for BAiSED AI Agents
## Guidelines
TypeScript is the primary language for BAiSED AI agent development on Base. Use TypeScript 5.x with strict mode enabled for all agent code, smart contract interactions, and API integrations. TypeScript provides type safety that prevents runtime errors, enables better IDE support, and makes code self-documenting.

When to use this skill:
- Building agent services that interact with Base blockchain contracts
- Implementing async workflows for transaction submission and monitoring
- Creating type-safe wrappers around viem or wagmi for contract calls
- Handling complex data structures from on-chain queries
- Developing NextJS-based agent interfaces with API routes
- Writing reusable utility functions with proper generic constraints

Key principles:
- Always enable strict mode in tsconfig.json (strict: true, strictNullChecks: true)
- Use type inference where possible but add explicit types for function signatures
- Leverage discriminated unions for state machines and result types
- Handle BigInt types correctly for on-chain numeric values
- Use async/await patterns with proper error handling for all blockchain operations
- Define typed ABIs and use viem's type inference for contract interactions
- Prefer unknown over any, use type guards for narrowing
- Use utility types (Pick, Omit, Partial, Required) to derive types

TypeScript provides essential safety for agents handling financial transactions and user funds. The type system catches errors at compile time that would otherwise cause runtime failures in production.

## Examples
Trigger: "Create a typed function to check reputation score"
Action: Use TypeScript with viem to create a fully typed contract read function:
```typescript
import { createPublicClient, http, Address } from 'viem';
import { base } from 'viem/chains';

const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63' as const;

type ReputationScore = {
  score: bigint;
  lastUpdated: bigint;
  isActive: boolean;
};

async function getReputationScore(agentAddress: Address): Promise<ReputationScore> {
  const client = createPublicClient({
    chain: base,
    transport: http('https://mainnet.base.org')
  });
  
  const score = await client.readContract({
    address: REPUTATION_REGISTRY,
    abi: [{
      name: 'getScore',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'agent', type: 'address' }],
      outputs: [{ name: '', type: 'uint256' }]
    }],
    functionName: 'getScore',
    args: [agentAddress]
  });
  
  return {
    score,
    lastUpdated: BigInt(Date.now()),
    isActive: score > 0n
  };
}
```

Trigger: "Handle multiple async contract calls with proper error handling"
Action: Use Promise.allSettled with TypeScript discriminated unions:
```typescript
type ContractCallResult<T> = 
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

async function fetchAgentData(addresses: Address[]): Promise<ContractCallResult<bigint>[]> {
  const results = await Promise.allSettled(
    addresses.map(addr => getReputationScore(addr))
  );
  
  return results.map(result => {
    if (result.status === 'fulfilled') {
      return { status: 'success', data: result.value.score };
    } else {
      return { status: 'error', error: result.reason };
    }
  });
}
```

## Resources
- TypeScript Handbook: https://www.typescriptlang.org/docs/handbook/intro.html
- TypeScript 5.x Release Notes: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html
- Viem TypeScript Guide: https://viem.sh/docs/typescript.html
- Base Contract Addresses: https://docs.base.org/base-contracts/

## Cross-References
- viem-wagmi: Typed contract interactions and wallet management
- nextjs: TypeScript in API routes and server components
- git: Managing TypeScript build artifacts and configuration files