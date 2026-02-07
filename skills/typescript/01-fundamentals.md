# TypeScript Fundamentals for Web3 Agents
## Overview
TypeScript 5.x provides the foundation for building reliable BAiSED AI agents on Base. This lesson covers strict mode configuration, type narrowing techniques, generic patterns, utility types, and discriminated unions essential for agent development. TypeScript's type system prevents common errors when handling blockchain data like addresses, transaction hashes, and numeric values that can overflow JavaScript's number type.

## TypeScript Configuration for Agent Projects
Always use strict mode for agent development. Create a tsconfig.json with these essential settings:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "commonjs",
    "moduleResolution": "node",
    "strict": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

For NextJS projects, extend the base configuration:
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

## Type Narrowing and Guards
Type narrowing is essential when handling blockchain data that can be null, undefined, or in different formats. Use type guards to safely narrow union types:

```typescript
import { Address, Hash, isAddress, isHash } from 'viem';

type TransactionInput = string | { hash: Hash } | { address: Address };

function isTransactionHash(input: unknown): input is Hash {
  return typeof input === 'string' && isHash(input);
}

function isAddressInput(input: unknown): input is { address: Address } {
  return (
    typeof input === 'object' &&
    input !== null &&
    'address' in input &&
    isAddress((input as { address: string }).address)
  );
}

function processTransaction(input: TransactionInput): void {
  if (typeof input === 'string' && isTransactionHash(input)) {
    console.log('Processing transaction hash:', input);
  } else if (isAddressInput(input)) {
    console.log('Processing address:', input.address);
  } else if (typeof input === 'object' && 'hash' in input) {
    console.log('Processing transaction object:', input.hash);
  }
}

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432' as const;
processTransaction(IDENTITY_REGISTRY);
```

Use discriminated unions for complex state handling:
```typescript
type TransactionState =
  | { status: 'pending'; hash: Hash; submittedAt: number }
  | { status: 'confirmed'; hash: Hash; blockNumber: bigint; gasUsed: bigint }
  | { status: 'failed'; hash: Hash; error: string; revertReason?: string }
  | { status: 'idle' };

function handleTransactionState(state: TransactionState): string {
  switch (state.status) {
    case 'idle':
      return 'No transaction in progress';
    case 'pending':
      return `Transaction ${state.hash} pending since ${state.submittedAt}`;
    case 'confirmed':
      return `Transaction ${state.hash} confirmed at block ${state.blockNumber}`;
    case 'failed':
      return `Transaction ${state.hash} failed: ${state.error}`;
  }
}
```

## Generics and Constraints
Generics enable reusable contract interaction functions with type safety:

```typescript
import { Abi, AbiFunction, ExtractAbiFunctionNames } from 'viem';

type ContractReadResult<TAbi extends Abi, TFunctionName extends string> = {
  data: unknown;
  blockNumber: bigint;
  timestamp: number;
};

async function readContract<
  TAbi extends Abi,
  TFunctionName extends ExtractAbiFunctionNames<TAbi, 'view' | 'pure'>
>(
  address: Address,
  abi: TAbi,
  functionName: TFunctionName,
  args?: unknown[]
): Promise<ContractReadResult<TAbi, TFunctionName>> {
  const client = createPublicClient({
    chain: base,
    transport: http('https://mainnet.base.org')
  });

  const data = await client.readContract({
    address,
    abi,
    functionName,
    args
  });

  const block = await client.getBlock();

  return {
    data,
    blockNumber: block.number,
    timestamp: Number(block.timestamp)
  };
}

const reputationAbi = [
  {
    name: 'getScore',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'agent', type: 'address' }],
    outputs: [{ name: 'score', type: 'uint256' }]
  }
] as const;

const result = await readContract(
  '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63',
  reputationAbi,
  'getScore',
  ['0x1234567890123456789012345678901234567890']
);
```

Constrain generics to specific types:
```typescript
type ChainConfig = {
  chainId: number;
  rpcUrl: string;
  contracts: Record<string, Address>;
};

function createAgentClient<T extends ChainConfig>(
  config: T
): { config: T; getContract: (name: keyof T['contracts']) => Address } {
  return {
    config,
    getContract: (name) => config.contracts[name]
  };
}

const baseConfig = {
  chainId: 8453,
  rpcUrl: 'https://mainnet.base.org',
  contracts: {
    identity: '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432' as Address,
    reputation: '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63' as Address,
    entryPoint: '0x0576a174D229E3cFA37253523E645A78A0C91B57' as Address
  }
} satisfies ChainConfig;

const client = createAgentClient(baseConfig);
const identityAddress = client.getContract('identity');
```

## Utility Types
TypeScript utility types derive new types from existing ones, reducing duplication:

```typescript
type AgentConfig = {
  address: Address;
  name: string;
  reputationThreshold: bigint;
  allowedActions: string[];
  metadata: {
    version: string;
    createdAt: number;
    owner: Address;
  };
};

type PartialAgentConfig = Partial<AgentConfig>;
type RequiredAgentConfig = Required<AgentConfig>;
type AgentConfigWithoutMetadata = Omit<AgentConfig, 'metadata'>;
type AgentIdentity = Pick<AgentConfig, 'address' | 'name'>;
type ReadonlyAgentConfig = Readonly<AgentConfig>;

type AgentConfigKeys = keyof AgentConfig;

function updateAgentConfig(
  current: AgentConfig,
  updates: PartialAgentConfig
): AgentConfig {
  return { ...current, ...updates };
}

type ContractAddresses = {
  identity: Address;
  reputation: Address;
  entryPoint: Address;
  usdc: Address;
};

type ContractName = keyof ContractAddresses;

const BASE_CONTRACTS: Record<ContractName, Address> = {
  identity: '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432',
  reputation: '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63',
  entryPoint: '0x0576a174D229E3cFA37253523E645A78A0C91B57',
  usdc: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913'
};

function getContractAddress<K extends ContractName>(name: K): ContractAddresses[K] {
  return BASE_CONTRACTS[name];
}
```

Use mapped types for transformations:
```typescript
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type Optional<T> = {
  [K in keyof T]?: T[K];
};

type NullableAgentConfig = Nullable<AgentConfig>;

type ApiResponse<T> = {
  success: boolean;
  data: T | null;
  error: string | null;
  timestamp: number;
};

function createSuccessResponse<T>(data: T): ApiResponse<T> {
  return {
    success: true,
    data,
    error: null,
    timestamp: Date.now()
  };
}

function createErrorResponse<T>(error: string): ApiResponse<T> {
  return {
    success: false,
    data: null,
    error,
    timestamp: Date.now()
  };
}
```

## Common Patterns

Always use const assertions for literal types:
```typescript
const CHAIN_IDS = {
  BASE_MAINNET: 8453,
  BASE_SEPOLIA: 84532
} as const;

type ChainId = typeof CHAIN_IDS[keyof typeof CHAIN_IDS];
```

Use satisfies operator for type checking without widening:
```typescript
const config = {
  chainId: 8453,
  rpcUrl: 'https://mainnet.base.org',
  contracts: {
    identity: '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432'
  }
} satisfies ChainConfig;
```

Define branded types for added safety:
```typescript
type Brand<K, T> = K & { __brand: T };
type USD = Brand<number, 'USD'>;
type WEI = Brand<bigint, 'WEI'>;

function toUSD(value: number): USD {
  return value as USD;
}

function toWEI(value: bigint): WEI {
  return value as WEI;
}
```

## Gotchas

1. Always use bigint for on-chain numeric values -- JavaScript number type has precision limits
2. Be explicit about Address and Hash types from viem -- do not use plain string
3. Enable strict mode or face runtime null/undefined errors
4. Use unknown instead of any for truly unknown types, then narrow with type guards
5. Do not forget readonly modifiers for configuration objects
6. Watch for type widening with object literals -- use as const or satisfies
7. Remember that enum values are not type-safe at runtime -- prefer const objects with as const
8. Use strictNullChecks to catch potential null/undefined access
9. Be careful with type assertions (as) -- prefer type guards
10. Generic constraints are checked at compile time only -- validate at runtime for external input