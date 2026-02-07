# Async Patterns for Web3 Agents
## Overview
Asynchronous programming is fundamental to BAiSED AI agents interacting with Base blockchain. All contract reads, transaction submissions, and RPC calls are async operations. This lesson covers Promises, async/await syntax, error handling patterns, retry logic with exponential backoff, concurrent execution strategies, and event-driven patterns for transaction monitoring.

## Promises and Async/Await
Every blockchain interaction returns a Promise. Use async/await for readable, sequential code:

```typescript
import { createPublicClient, http, Address, parseEther } from 'viem';
import { base } from 'viem/chains';

const client = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

async function getAgentBalance(address: Address): Promise<bigint> {
  try {
    const balance = await client.getBalance({ address });
    return balance;
  } catch (error) {
    if (error instanceof Error) {
      throw new Error(`Failed to fetch balance: ${error.message}`);
    }
    throw error;
  }
}

async function checkMultipleBalances(addresses: Address[]): Promise<Map<Address, bigint>> {
  const balances = new Map<Address, bigint>();
  
  for (const address of addresses) {
    const balance = await getAgentBalance(address);
    balances.set(address, balance);
  }
  
  return balances;
}

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432' as const;
const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63' as const;

async function initializeAgent(): Promise<void> {
  const identityBalance = await getAgentBalance(IDENTITY_REGISTRY);
  console.log(`Identity Registry balance: ${identityBalance}`);
  
  const reputationBalance = await getAgentBalance(REPUTATION_REGISTRY);
  console.log(`Reputation Registry balance: ${reputationBalance}`);
}
```

Chain promises for sequential operations:
```typescript
async function registerAndVerifyAgent(agentAddress: Address): Promise<boolean> {
  const registered = await registerAgent(agentAddress)
    .then(txHash => waitForTransaction(txHash))
    .then(receipt => receipt.status === 'success')
    .catch(error => {
      console.error('Registration failed:', error);
      return false;
    });
  
  if (!registered) {
    return false;
  }
  
  const verified = await verifyAgentIdentity(agentAddress);
  return verified;
}

async function registerAgent(address: Address): Promise<Hash> {
  // Simulated registration
  return '0x1234567890123456789012345678901234567890123456789012345678901234' as Hash;
}

async function waitForTransaction(hash: Hash): Promise<{ status: 'success' | 'reverted' }> {
  const receipt = await client.waitForTransactionReceipt({ hash });
  return { status: receipt.status };
}

async function verifyAgentIdentity(address: Address): Promise<boolean> {
  const isRegistered = await client.readContract({
    address: IDENTITY_REGISTRY,
    abi: [{
      name: 'isRegistered',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'agent', type: 'address' }],
      outputs: [{ name: '', type: 'bool' }]
    }],
    functionName: 'isRegistered',
    args: [address]
  });
  return isRegistered as boolean;
}
```

## Error Handling Patterns
Implement comprehensive error handling for blockchain operations:

```typescript
type BlockchainError = 
  | { type: 'RPC_ERROR'; message: string; code?: number }
  | { type: 'CONTRACT_ERROR'; message: string; revertReason?: string }
  | { type: 'NETWORK_ERROR'; message: string }
  | { type: 'TIMEOUT_ERROR'; message: string };

class AgentError extends Error {
  constructor(
    message: string,
    public readonly blockchainError: BlockchainError
  ) {
    super(message);
    this.name = 'AgentError';
  }
}

function isRPCError(error: unknown): error is { code: number; message: string } {
  return (
    typeof error === 'object' &&
    error !== null &&
    'code' in error &&
    'message' in error
  );
}

async function safeReadContract<T>(
  address: Address,
  abi: unknown[],
  functionName: string,
  args?: unknown[]
): Promise<T> {
  try {
    const result = await client.readContract({
      address,
      abi,
      functionName,
      args
    });
    return result as T;
  } catch (error) {
    if (isRPCError(error)) {
      throw new AgentError('RPC call failed', {
        type: 'RPC_ERROR',
        message: error.message,
        code: error.code
      });
    }
    
    if (error instanceof Error) {
      if (error.message.includes('reverted')) {
        throw new AgentError('Contract call reverted', {
          type: 'CONTRACT_ERROR',
          message: error.message,
          revertReason: extractRevertReason(error.message)
        });
      }
      
      if (error.message.includes('network')) {
        throw new AgentError('Network error', {
          type: 'NETWORK_ERROR',
          message: error.message
        });
      }
    }
    
    throw new AgentError('Unknown error', {
      type: 'RPC_ERROR',
      message: String(error)
    });
  }
}

function extractRevertReason(message: string): string | undefined {
  const match = message.match(/reason: (.+)/);
  return match?.[1];
}

async function getReputationWithFallback(address: Address): Promise<bigint> {
  try {
    return await safeReadContract<bigint>(
      REPUTATION_REGISTRY,
      [{
        name: 'getScore',
        type: 'function',
        stateMutability: 'view',
        inputs: [{ name: 'agent', type: 'address' }],
        outputs: [{ name: '', type: 'uint256' }]
      }],
      'getScore',
      [address]
    );
  } catch (error) {
    if (error instanceof AgentError && error.blockchainError.type === 'CONTRACT_ERROR') {
      console.warn('Contract read failed, returning default score');
      return 0n;
    }
    throw error;
  }
}
```

## Retry Logic with Exponential Backoff
Implement robust retry logic for transient failures:

```typescript
type RetryOptions = {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  backoffMultiplier: number;
  retryableErrors?: string[];
};

const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxRetries: 3,
  initialDelay: 1000,
  maxDelay: 10000,
  backoffMultiplier: 2,
  retryableErrors: ['network', 'timeout', 'rate limit']
};

function isRetryableError(error: unknown, retryableErrors: string[]): boolean {
  if (!(error instanceof Error)) {
    return false;
  }
  
  const message = error.message.toLowerCase();
  return retryableErrors.some(retryable => message.includes(retryable));
}

function calculateDelay(attempt: number, options: RetryOptions): number {
  const delay = options.initialDelay * Math.pow(options.backoffMultiplier, attempt);
  return Math.min(delay, options.maxDelay);
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function withRetry<T>(
  operation: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_RETRY_OPTIONS, ...options };
  let lastError: unknown;
  
  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      
      if (attempt === opts.maxRetries) {
        break;
      }
      
      if (!isRetryableError(error, opts.retryableErrors || [])) {
        throw error;
      }
      
      const delay = calculateDelay(attempt, opts);
      console.log(`Retry attempt ${attempt + 1}/${opts.maxRetries} after ${delay}ms`);
      await sleep(delay);
    }
  }
  
  throw lastError;
}

async function fetchBlockNumberWithRetry(): Promise<bigint> {
  return withRetry(
    async () => {
      const blockNumber = await client.getBlockNumber();
      return blockNumber;
    },
    {
      maxRetries: 5,
      initialDelay: 500,
      retryableErrors: ['network', 'timeout', 'rate']
    }
  );
}

async function submitTransactionWithRetry(
  to: Address,
  data: `0x${string}`
): Promise<Hash> {
  return withRetry(
    async () => {
      // Simulated transaction submission
      const hash = '0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890' as Hash;
      return hash;
    },
    {
      maxRetries: 3,
      initialDelay: 2000,
      retryableErrors: ['replacement transaction underpriced', 'nonce too low']
    }
  );
}
```

## Concurrent Execution
Use Promise.all and Promise.allSettled for parallel operations:

```typescript
async function fetchAgentDataConcurrently(
  addresses: Address[]
): Promise<Array<{ address: Address; balance: bigint; reputation: bigint }>> {
  const results = await Promise.all(
    addresses.map(async address => {
      const [balance, reputation] = await Promise.all([
        getAgentBalance(address),
        getReputationWithFallback(address)
      ]);
      
      return { address, balance, reputation };
    })
  );
  
  return results;
}

type SettledResult<T> = 
  | { status: 'fulfilled'; value: T }
  | { status: 'rejected'; reason: Error };

async function fetchAgentDataSafely(
  addresses: Address[]
): Promise<SettledResult<{ address: Address; balance: bigint; reputation: bigint }>[]> {
  const promises = addresses.map(async address => {
    const [balance, reputation] = await Promise.all([
      getAgentBalance(address),
      getReputationWithFallback(address)
    ]);
    return { address, balance, reputation };
  });
  
  const results = await Promise.allSettled(promises);
  return results as SettledResult<{ address: Address; balance: bigint; reputation: bigint }>[];
}

async function processSettledResults(
  addresses: Address[]
): Promise<void> {
  const results = await fetchAgentDataSafely(addresses);
  
  const successful = results.filter(
    (r): r is { status: 'fulfilled'; value: { address: Address; balance: bigint; reputation: bigint } } =>
      r.status === 'fulfilled'
  );
  
  const failed = results.filter(
    (r): r is { status: 'rejected'; reason: Error } =>
      r.status === 'rejected'
  );
  
  console.log(`Successfully fetched ${successful.length} agents`);
  console.log(`Failed to fetch ${failed.length} agents`);
  
  for (const result of failed) {
    console.error('Error:', result.reason.message);
  }
}

async function raceForFastestRPC(urls: string[]): Promise<bigint> {
  const promises = urls.map(url => {
    const rpcClient = createPublicClient({
      chain: base,
      transport: http(url)
    });
    return rpcClient.getBlockNumber();
  });
  
  return Promise.race(promises);
}
```

## Event-Driven Patterns
Monitor blockchain events and transactions asynchronously:

```typescript
import { Log, WatchContractEventReturnType } from 'viem';

type EventCallback<T> = (event: T) => void | Promise<void>;

class EventMonitor {
  private unwatch: WatchContractEventReturnType | null = null;
  
  async watchReputationUpdates(
    callback: EventCallback<{ agent: Address; newScore: bigint }>
  ): Promise<void> {
    this.unwatch = client.watchContractEvent({
      address: REPUTATION_REGISTRY,
      abi: [{
        name: 'ScoreUpdated',
        type: 'event',
        inputs: [
          { name: 'agent', type: 'address', indexed: true },
          { name: 'newScore', type: 'uint256', indexed: false }
        ]
      }],
      eventName: 'ScoreUpdated',
      onLogs: async (logs) => {
        for (const log of logs) {
          const { agent, newScore } = log.args as { agent: Address; newScore: bigint };
          await callback({ agent, newScore });
        }
      }
    });
  }
  
  stop(): void {
    if (this.unwatch) {
      this.unwatch();
      this.unwatch = null;
    }
  }
}

async function monitorAgentRegistrations(): Promise<void> {
  const monitor = new EventMonitor();
  
  await monitor.watchReputationUpdates(async ({ agent, newScore }) => {
    console.log(`Agent ${agent} score updated to ${newScore}`);
    
    if (newScore > 1000n) {
      console.log('High reputation agent detected');
    }
  });
  
  await sleep(60000);
  monitor.stop();
}

async function waitForTransactionWithTimeout(
  hash: Hash,
  timeoutMs: number = 30000
): Promise<{ status: 'success' | 'reverted' }> {
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error('Transaction timeout')), timeoutMs);
  });
  
  const receiptPromise = client.waitForTransactionReceipt({ hash });
  
  try {
    const receipt = await Promise.race([receiptPromise, timeoutPromise]);
    return { status: receipt.status };
  } catch (error) {
    if (error instanceof Error && error.message === 'Transaction timeout') {
      throw new AgentError('Transaction timed out', {
        type: 'TIMEOUT_ERROR',
        message: `Transaction ${hash} not confirmed within ${timeoutMs}ms`
      });
    }
    throw error;
  }
}
```

## Common Patterns

Always handle errors explicitly in async functions:
```typescript
async function safeOperation(): Promise<void> {
  try {
    await riskyOperation();
  } catch (error) {
    console.error('Operation failed:', error);
    throw error;
  }
}
```

Use AbortController for cancellable operations:
```typescript
async function fetchWithTimeout(url: string, timeoutMs: number): Promise<Response> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } finally {
    clearTimeout(timeout);
  }
}
```

## Gotchas

1. Always await promises in async functions -- forgotten awaits lead to unhandled rejections
2. Use Promise.allSettled when you need all results regardless of failures
3. Never use async without proper error handling -- uncaught errors crash the process
4. Be careful with concurrent writes -- use transaction queues for sequential execution
5. Set reasonable timeouts for all blockchain operations
6. Remember that Promise.race rejects if any promise rejects
7. Clean up event listeners and watchers to prevent memory leaks
8. Use exponential backoff to avoid overwhelming RPC endpoints
9. Handle nonce conflicts when submitting multiple transactions concurrently
10. Test timeout and error scenarios explicitly