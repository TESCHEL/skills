# Web3 TypeScript with Viem
## Overview
TypeScript integration with viem provides complete type safety for smart contract interactions on Base. This lesson covers typed contract reads and writes, ABI type inference, BigInt handling for on-chain numeric values, hex string types, and comparisons between ethers.js and viem approaches. Proper typing prevents common Web3 errors like incorrect function arguments, mishandled numeric precision, and invalid address formats.

## Typed Contract Interactions with Viem
Viem provides first-class TypeScript support with full ABI type inference:

```typescript
import { createPublicClient, createWalletClient, http, Address, parseAbi } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432' as const;
const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63' as const;
const USDC = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913' as const;

const identityAbi = parseAbi([
  'function isRegistered(address agent) view returns (bool)',
  'function getIdentity(address agent) view returns (string name, uint256 registeredAt, bool active)',
  'function register(string calldata name) external',
  'event Registered(address indexed agent, string name, uint256 timestamp)'
]);

const reputationAbi = parseAbi([
  'function getScore(address agent) view returns (uint256)',
  'function updateScore(address agent, uint256 newScore) external',
  'function getHistory(address agent) view returns (uint256[] scores, uint256[] timestamps)',
  'event ScoreUpdated(address indexed agent, uint256 newScore, uint256 timestamp)'
]);

async function checkAgentRegistration(agentAddress: Address): Promise<boolean> {
  const isRegistered = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: identityAbi,
    functionName: 'isRegistered',
    args: [agentAddress]
  });
  
  return isRegistered;
}

async function getAgentIdentity(agentAddress: Address): Promise<{
  name: string;
  registeredAt: bigint;
  active: boolean;
}> {
  const identity = await publicClient.readContract({
    address: IDENTITY_REGISTRY,
    abi: identityAbi,
    functionName: 'getIdentity',
    args: [agentAddress]
  });
  
  return {
    name: identity[0],
    registeredAt: identity[1],
    active: identity[2]
  };
}

async function getReputationScore(agentAddress: Address): Promise<bigint> {
  const score = await publicClient.readContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'getScore',
    args: [agentAddress]
  });
  
  return score;
}

async function getReputationHistory(agentAddress: Address): Promise<{
  scores: readonly bigint[];
  timestamps: readonly bigint[];
}> {
  const history = await publicClient.readContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'getHistory',
    args: [agentAddress]
  });
  
  return {
    scores: history[0],
    timestamps: history[1]
  };
}
```

Write operations with typed transactions:
```typescript
import { Hash, TransactionReceipt } from 'viem';

const account = privateKeyToAccount('0x...' as `0x${string}`);

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
});

async function registerAgent(name: string): Promise<Hash> {
  const hash = await walletClient.writeContract({
    address: IDENTITY_REGISTRY,
    abi: identityAbi,
    functionName: 'register',
    args: [name]
  });
  
  return hash;
}

async function registerAndWait(name: string): Promise<TransactionReceipt> {
  const hash = await registerAgent(name);
  
  const receipt = await publicClient.waitForTransactionReceipt({
    hash,
    confirmations: 2
  });
  
  return receipt;
}

async function updateAgentScore(
  agentAddress: Address,
  newScore: bigint
): Promise<Hash> {
  const hash = await walletClient.writeContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'updateScore',
    args: [agentAddress, newScore]
  });
  
  return hash;
}

async function simulateScoreUpdate(
  agentAddress: Address,
  newScore: bigint
): Promise<void> {
  const { request } = await publicClient.simulateContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'updateScore',
    args: [agentAddress, newScore],
    account
  });
  
  console.log('Simulation successful, request:', request);
}
```

## ABI Type Inference
Viem infers types from ABIs automatically:

```typescript
import { Abi, ExtractAbiFunctionNames, AbiParametersToPrimitiveTypes } from 'viem';

const usdcAbi = [
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'transfer',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'to', type: 'address' },
      { name: 'amount', type: 'uint256' }
    ],
    outputs: [{ name: '', type: 'bool' }]
  },
  {
    name: 'allowance',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' }
    ],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'approve',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' }
    ],
    outputs: [{ name: '', type: 'bool' }]
  },
  {
    name: 'Transfer',
    type: 'event',
    inputs: [
      { name: 'from', type: 'address', indexed: true },
      { name: 'to', type: 'address', indexed: true },
      { name: 'value', type: 'uint256', indexed: false }
    ]
  }
] as const satisfies Abi;

type USDCFunctions = ExtractAbiFunctionNames<typeof usdcAbi>;

async function getUSDCBalance(userAddress: Address): Promise<bigint> {
  const balance = await publicClient.readContract({
    address: USDC,
    abi: usdcAbi,
    functionName: 'balanceOf',
    args: [userAddress]
  });
  
  return balance;
}

async function transferUSDC(
  to: Address,
  amount: bigint
): Promise<Hash> {
  const hash = await walletClient.writeContract({
    address: USDC,
    abi: usdcAbi,
    functionName: 'transfer',
    args: [to, amount]
  });
  
  return hash;
}

async function approveUSDC(
  spender: Address,
  amount: bigint
): Promise<Hash> {
  const hash = await walletClient.writeContract({
    address: USDC,
    abi: usdcAbi,
    functionName: 'approve',
    args: [spender, amount]
  });
  
  return hash;
}

async function checkAllowance(
  owner: Address,
  spender: Address
): Promise<bigint> {
  const allowance = await publicClient.readContract({
    address: USDC,
    abi: usdcAbi,
    functionName: 'allowance',
    args: [owner, spender]
  });
  
  return allowance;
}
```

Create generic typed wrappers:
```typescript
class TypedContract<TAbi extends Abi> {
  constructor(
    private address: Address,
    private abi: TAbi,
    private publicClient: typeof publicClient,
    private walletClient: typeof walletClient
  ) {}
  
  async read<TFunctionName extends ExtractAbiFunctionNames<TAbi, 'view' | 'pure'>>(
    functionName: TFunctionName,
    args?: unknown[]
  ): Promise<unknown> {
    return this.publicClient.readContract({
      address: this.address,
      abi: this.abi,
      functionName,
      args
    });
  }
  
  async write<TFunctionName extends ExtractAbiFunctionNames<TAbi, 'nonpayable' | 'payable'>>(
    functionName: TFunctionName,
    args?: unknown[],
    value?: bigint
  ): Promise<Hash> {
    return this.walletClient.writeContract({
      address: this.address,
      abi: this.abi,
      functionName,
      args,
      value
    });
  }
}

const usdcContract = new TypedContract(
  USDC,
  usdcAbi,
  publicClient,
  walletClient
);

const balance = await usdcContract.read('balanceOf', ['0x1234567890123456789012345678901234567890']);
```

## BigInt Handling
All on-chain numeric values use BigInt in TypeScript:

```typescript
import { formatUnits, parseUnits } from 'viem';

function formatUSDC(amount: bigint): string {
  return formatUnits(amount, 6);
}

function parseUSDC(amount: string): bigint {
  return parseUnits(amount, 6);
}

function formatETH(amount: bigint): string {
  return formatUnits(amount, 18);
}

function parseETH(amount: string): bigint {
  return parseUnits(amount, 18);
}

const tenUSDC = parseUSDC('10.0');
const oneETH = parseETH('1.0');

console.log('10 USDC in base units:', tenUSDC);
console.log('1 ETH in wei:', oneETH);

function calculatePercentage(amount: bigint, percentage: number): bigint {
  const basisPoints = BigInt(Math.floor(percentage * 100));
  return (amount * basisPoints) / 10000n;
}

function addBigInts(a: bigint, b: bigint): bigint {
  return a + b;
}

function multiplyBigInts(a: bigint, b: bigint): bigint {
  return a * b;
}

function divideBigInts(a: bigint, b: bigint): bigint {
  if (b === 0n) {
    throw new Error('Division by zero');
  }
  return a / b;
}

function compareBigInts(a: bigint, b: bigint): -1 | 0 | 1 {
  if (a < b) return -1;
  if (a > b) return 1;
  return 0;
}

const balance1 = parseUSDC('100.50');
const balance2 = parseUSDC('50.25');
const total = addBigInts(balance1, balance2);
console.log('Total balance:', formatUSDC(total), 'USDC');

const fee = calculatePercentage(total, 0.3);
console.log('0.3% fee:', formatUSDC(fee), 'USDC');

type TokenAmount = {
  value: bigint;
  decimals: number;
  symbol: string;
};

function formatTokenAmount(token: TokenAmount): string {
  const formatted = formatUnits(token.value, token.decimals);
  return `${formatted} ${token.symbol}`;
}

const usdcAmount: TokenAmount = {
  value: parseUSDC('1000.00'),
  decimals: 6,
  symbol: 'USDC'
};

console.log('Amount:', formatTokenAmount(usdcAmount));
```

## Hex String Types
Viem enforces hex string types for addresses, hashes, and data:

```typescript
import { Hex, isAddress, isHex, toHex, fromHex, keccak256, encodeFunctionData } from 'viem';

function validateAddress(address: string): Address {
  if (!isAddress(address)) {
    throw new Error(`Invalid address: ${address}`);
  }
  return address;
}

function validateHash(hash: string): Hash {
  if (!isHex(hash) || hash.length !== 66) {
    throw new Error(`Invalid transaction hash: ${hash}`);
  }
  return hash as Hash;
}

function stringToHex(str: string): Hex {
  return toHex(str);
}

function hexToString(hex: Hex): string {
  return fromHex(hex, 'string');
}

function numberToHex(num: number | bigint): Hex {
  return toHex(num);
}

function hexToNumber(hex: Hex): bigint {
  return fromHex(hex, 'bigint');
}

const message = 'Hello Base';
const messageHex = stringToHex(message);
console.log('Message as hex:', messageHex);

const messageHash = keccak256(messageHex);
console.log('Message hash:', messageHash);

const encodedData = encodeFunctionData({
  abi: identityAbi,
  functionName: 'register',
  args: ['MyAgent']
});

console.log('Encoded function call:', encodedData);

type TypedHex<T extends string> = `0x${string}` & { __type: T };

type AddressHex = TypedHex<'address'>;
type HashHex = TypedHex<'hash'>;
type DataHex = TypedHex<'data'>;

function createTypedHex<T extends string>(hex: Hex): TypedHex<T> {
  return hex as TypedHex<T>;
}
```

## Ethers.js vs Viem Comparison
Key differences between ethers.js and viem approaches:

```typescript
import { JsonRpcProvider, Contract, Wallet } from 'ethers';

const ethersProvider = new JsonRpcProvider('https://mainnet.base.org');
const ethersWallet = new Wallet('0x...', ethersProvider);

const ethersContract = new Contract(
  REPUTATION_REGISTRY,
  [
    'function getScore(address agent) view returns (uint256)',
    'function updateScore(address agent, uint256 newScore)'
  ],
  ethersWallet
);

async function ethersGetScore(agentAddress: string): Promise<bigint> {
  const score = await ethersContract.getScore(agentAddress);
  return score;
}

async function ethersUpdateScore(
  agentAddress: string,
  newScore: bigint
): Promise<string> {
  const tx = await ethersContract.updateScore(agentAddress, newScore);
  const receipt = await tx.wait();
  return receipt.hash;
}

async function viemGetScore(agentAddress: Address): Promise<bigint> {
  return publicClient.readContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'getScore',
    args: [agentAddress]
  });
}

async function viemUpdateScore(
  agentAddress: Address,
  newScore: bigint
): Promise<Hash> {
  const hash = await walletClient.writeContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'updateScore',
    args: [agentAddress, newScore]
  });
  
  await publicClient.waitForTransactionReceipt({ hash });
  return hash;
}
```

Viem advantages:
- Full TypeScript type inference from ABIs
- Smaller bundle size and tree-shakeable
- More explicit client separation (public vs wallet)
- Better BigInt handling without conversion
- Stricter type safety for addresses and hashes
- No class-based abstractions

Ethers.js advantages:
- Larger ecosystem and community
- More established documentation
- Compatible with older projects
- Built-in ENS resolution

## Common Patterns

Always define ABIs with as const:
```typescript
const abi = [
  'function foo() view returns (uint256)'
] as const;
```

Use multicall for batch reads:
```typescript
import { multicall } from 'viem/actions';

const results = await publicClient.multicall({
  contracts: [
    {
      address: USDC,
      abi: usdcAbi,
      functionName: 'balanceOf',
      args: [account.address]
    },
    {
      address: REPUTATION_REGISTRY,
      abi: reputationAbi,
      functionName: 'getScore',
      args: [account.address]
    }
  ]
});
```

Handle contract errors explicitly:
```typescript
import { ContractFunctionExecutionError } from 'viem';

try {
  await walletClient.writeContract({
    address: REPUTATION_REGISTRY,
    abi: reputationAbi,
    functionName: 'updateScore',
    args: [account.address, 1000n]
  });
} catch (error) {
  if (error instanceof ContractFunctionExecutionError) {
    console.error('Contract execution failed:', error.message);
  }
  throw error;
}
```

## Gotchas

1. Always use BigInt for on-chain numeric values -- number type loses precision
2. Viem requires exact type matches -- use as const for ABIs
3. Address and Hash types are branded -- cannot use plain strings
4. parseUnits and formatUnits require correct decimal places
5. Multicall results can be errors -- check status before accessing result
6. Contract function names are case-sensitive in ABIs
7. Viem does not auto-convert between number and bigint
8. Watch for integer division truncation with BigInt
9. Always validate addresses before using them in contract calls
10. Use parseAbi for human-readable ABIs, full ABI objects for complex types