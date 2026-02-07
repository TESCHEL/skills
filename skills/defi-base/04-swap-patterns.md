# Common DeFi Swap Patterns on Base

## Overview

This lesson covers production-ready patterns for token swaps and DeFi operations on Base. Topics include:

- Approve-and-swap pattern with optimization
- Multicall for atomic operations
- Slippage protection strategies
- Deadline handling
- WETH wrapping and unwrapping
- Aggregator integration (1inch, Paraswap)
- Gas optimization techniques
- Error handling and recovery

These patterns apply across Aerodrome, Uniswap, and other DEXes on Base.

## Approve-and-Swap Pattern

The standard two-step process for token swaps:

```typescript
import { createPublicClient, createWalletClient, http, parseUnits, formatUnits, maxUint256 } from 'viem';
import { base } from 'viem/chains';

const USDC = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const WETH = '0x4200000000000000000000000000000000000006';
const AERODROME_ROUTER = '0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43';

const erc20Abi = [
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
  }
] as const;

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

async function approveIfNeeded(
  walletClient: any,
  token: `0x${string}`,
  spender: `0x${string}`,
  amount: bigint
) {
  const account = walletClient.account;
  
  // Check current allowance
  const currentAllowance = await publicClient.readContract({
    address: token,
    abi: erc20Abi,
    functionName: 'allowance',
    args: [account.address, spender]
  });
  
  console.log(`Current allowance: ${currentAllowance}`);
  console.log(`Required amount: ${amount}`);
  
  if (currentAllowance >= amount) {
    console.log('Sufficient allowance, skipping approval');
    return null;
  }
  
  // Approve max uint256 for gas optimization (one-time approval)
  console.log('Approving maximum amount for future swaps...');
  const approveTx = await walletClient.writeContract({
    address: token,
    abi: erc20Abi,
    functionName: 'approve',
    args: [spender, maxUint256]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: approveTx });
  console.log('Approval confirmed:', approveTx);
  
  return approveTx;
}

async function executeSwap(
  walletClient: any,
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  amountIn: bigint,
  slippageBps: number = 50
) {
  // Step 1: Approve if needed
  await approveIfNeeded(walletClient, tokenIn, AERODROME_ROUTER, amountIn);
  
  // Step 2: Get quote and execute swap
  const routerAbi = [
    {
      name: 'getAmountsOut',
      type: 'function',
      stateMutability: 'view',
      inputs: [
        { name: 'amountIn', type: 'uint256' },
        { name: 'routes', type: 'tuple[]', components: [
          { name: 'from', type: 'address' },
          { name: 'to', type: 'address' },
          { name: 'stable', type: 'bool' },
          { name: 'factory', type: 'address' }
        ]}
      ],
      outputs: [{ name: 'amounts', type: 'uint256[]' }]
    },
    {
      name: 'swapExactTokensForTokens',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'amountIn', type: 'uint256' },
        { name: 'amountOutMin', type: 'uint256' },
        { name: 'routes', type: 'tuple[]', components: [
          { name: 'from', type: 'address' },
          { name: 'to', type: 'address' },
          { name: 'stable', type: 'bool' },
          { name: 'factory', type: 'address' }
        ]},
        { name: 'to', type: 'address' },
        { name: 'deadline', type: 'uint256' }
      ],
      outputs: [{ name: 'amounts', type: 'uint256[]' }]
    }
  ] as const;
  
  const routes = [{
    from: tokenIn,
    to: tokenOut,
    stable: false,
    factory: '0x420DD381b31aEf6683db6B902084cB0FFECe40Da'
  }];
  
  const amounts = await publicClient.readContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'getAmountsOut',
    args: [amountIn, routes]
  });
  
  const expectedOut = amounts[1];
  const minOut = (expectedOut * BigInt(10000 - slippageBps)) / BigInt(10000);
  
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 1200);
  
  const swapTx = await walletClient.writeContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'swapExactTokensForTokens',
    args: [amountIn, minOut, routes, walletClient.account.address, deadline]
  });
  
  return swapTx;
}
```

## Multicall for Atomic Operations

Batch multiple operations into a single transaction:

```typescript
interface Call {
  target: `0x${string}`;
  callData: `0x${string}`;
  allowFailure: boolean;
}

const multicallAbi = [
  {
    name: 'aggregate3',
    type: 'function',
    stateMutability: 'payable',
    inputs: [{
      name: 'calls',
      type: 'tuple[]',
      components: [
        { name: 'target', type: 'address' },
        { name: 'allowFailure', type: 'bool' },
        { name: 'callData', type: 'bytes' }
      ]
    }],
    outputs: [{
      name: 'returnData',
      type: 'tuple[]',
      components: [
        { name: 'success', type: 'bool' },
        { name: 'returnData', type: 'bytes' }
      ]
    }]
  }
] as const;

const MULTICALL3 = '0xcA11bde05977b3631167028862bE2a173976CA11';

async function approveAndSwapAtomic(
  walletClient: any,
  tokenIn: `0x${string}`,
  amountIn: bigint,
  routes: any[],
  minOut: bigint
) {
  const account = walletClient.account;
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 1200);
  
  // Encode approval call
  const approveCalldata = encodeFunctionData({
    abi: erc20Abi,
    functionName: 'approve',
    args: [AERODROME_ROUTER, maxUint256]
  });
  
  // Encode swap call
  const routerAbi = [
    {
      name: 'swapExactTokensForTokens',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'amountIn', type: 'uint256' },
        { name: 'amountOutMin', type: 'uint256' },
        { name: 'routes', type: 'tuple[]', components: [
          { name: 'from', type: 'address' },
          { name: 'to', type: 'address' },
          { name: 'stable', type: 'bool' },
          { name: 'factory', type: 'address' }
        ]},
        { name: 'to', type: 'address' },
        { name: 'deadline', type: 'uint256' }
      ],
      outputs: [{ name: 'amounts', type: 'uint256[]' }]
    }
  ] as const;
  
  const swapCalldata = encodeFunctionData({
    abi: routerAbi,
    functionName: 'swapExactTokensForTokens',
    args: [amountIn, minOut, routes, account.address, deadline]
  });
  
  // Create multicall
  const calls = [
    {
      target: tokenIn,
      allowFailure: false,
      callData: approveCalldata
    },
    {
      target: AERODROME_ROUTER,
      allowFailure: false,
      callData: swapCalldata
    }
  ];
  
  // Execute multicall
  const multicallTx = await walletClient.writeContract({
    address: MULTICALL3,
    abi: multicallAbi,
    functionName: 'aggregate3',
    args: [calls]
  });
  
  console.log('Atomic approve+swap executed:', multicallTx);
  return multicallTx;
}

import { encodeFunctionData } from 'viem';
```

## Dynamic Slippage Calculation

Adjust slippage based on trade size and volatility:

```typescript
function calculateDynamicSlippage(
  amountIn: bigint,
  poolLiquidity: bigint,
  baseSlippageBps: number = 50,
  volatilityMultiplier: number = 1.0
): number {
  // Calculate trade size as percentage of pool
  const tradePercentage = Number(amountIn) / Number(poolLiquidity);
  
  // Increase slippage for larger trades
  let dynamicSlippage = baseSlippageBps;
  
  if (tradePercentage > 0.01) { // > 1% of pool
    dynamicSlippage = baseSlippageBps * 2;
  }
  if (tradePercentage > 0.05) { // > 5% of pool
    dynamicSlippage = baseSlippageBps * 4;
  }
  if (tradePercentage > 0.10) { // > 10% of pool
    dynamicSlippage = baseSlippageBps * 8;
  }
  
  // Apply volatility multiplier
  dynamicSlippage = Math.floor(dynamicSlippage * volatilityMultiplier);
  
  // Cap at reasonable maximum (5%)
  return Math.min(dynamicSlippage, 500);
}

async function getPoolLiquidity(
  tokenA: `0x${string}`,
  tokenB: `0x${string}`,
  stable: boolean
): Promise<bigint> {
  const factoryAbi = [
    {
      name: 'getPair',
      type: 'function',
      stateMutability: 'view',
      inputs: [
        { name: 'tokenA', type: 'address' },
        { name: 'tokenB', type: 'address' },
        { name: 'stable', type: 'bool' }
      ],
      outputs: [{ name: 'pair', type: 'address' }]
    }
  ] as const;
  
  const pairAbi = [
    {
      name: 'getReserves',
      type: 'function',
      stateMutability: 'view',
      inputs: [],
      outputs: [
        { name: 'reserve0', type: 'uint256' },
        { name: 'reserve1', type: 'uint256' },
        { name: 'blockTimestampLast', type: 'uint256' }
      ]
    }
  ] as const;
  
  const factory = '0x420DD381b31aEf6683db6B902084cB0FFECe40Da';
  
  const pairAddress = await publicClient.readContract({
    address: factory,
    abi: factoryAbi,
    functionName: 'getPair',
    args: [tokenA, tokenB, stable]
  });
  
  const reserves = await publicClient.readContract({
    address: pairAddress,
    abi: pairAbi,
    functionName: 'getReserves'
  });
  
  // Return smaller reserve as proxy for liquidity depth
  return reserves[0] < reserves[1] ? reserves[0] : reserves[1];
}
```

## Deadline Management

Properly handle transaction deadlines:

```typescript
function getDeadline(minutesFromNow: number = 20): bigint {
  return BigInt(Math.floor(Date.now() / 1000) + (minutesFromNow * 60));
}

function isDeadlineExpired(deadline: bigint): boolean {
  const now = BigInt(Math.floor(Date.now() / 1000));
  return now >= deadline;
}

async function swapWithDeadlineRetry(
  walletClient: any,
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  amountIn: bigint,
  minOut: bigint,
  routes: any[],
  maxRetries: number = 3
) {
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      const deadline = getDeadline(20); // 20 minutes
      
      const routerAbi = [
        {
          name: 'swapExactTokensForTokens',
          type: 'function',
          stateMutability: 'nonpayable',
          inputs: [
            { name: 'amountIn', type: 'uint256' },
            { name: 'amountOutMin', type: 'uint256' },
            { name: 'routes', type: 'tuple[]', components: [
              { name: 'from', type: 'address' },
              { name: 'to', type: 'address' },
              { name: 'stable', type: 'bool' },
              { name: 'factory', type: 'address' }
            ]},
            { name: 'to', type: 'address' },
            { name: 'deadline', type: 'uint256' }
          ],
          outputs: [{ name: 'amounts', type: 'uint256[]' }]
        }
      ] as const;
      
      const swapTx = await walletClient.writeContract({
        address: AERODROME_ROUTER,
        abi: routerAbi,
        functionName: 'swapExactTokensForTokens',
        args: [amountIn, minOut, routes, walletClient.account.address, deadline]
      });
      
      const receipt = await publicClient.waitForTransactionReceipt({ 
        hash: swapTx,
        timeout: 60000 // 60 second timeout
      });
      
      if (receipt.status === 'success') {
        console.log('Swap successful:', swapTx);
        return swapTx;
      } else {
        throw new Error('Transaction reverted');
      }
    } catch (error: any) {
      console.error(`Swap attempt ${attempt + 1} failed:`, error.message);
      
      if (error.message.includes('deadline') || error.message.includes('expired')) {
        console.log('Deadline expired, retrying with new deadline...');
        attempt++;
        continue;
      }
      
      // Non-deadline error, throw immediately
      throw error;
    }
  }
  
  throw new Error(`Swap failed after ${maxRetries} attempts`);
}
```

## WETH Wrapping and Unwrapping

Handle ETH <-> WETH conversions:

```typescript
const wethAbi = [
  {
    name: 'deposit',
    type: 'function',
    stateMutability: 'payable',
    inputs: [],
    outputs: []
  },
  {
    name: 'withdraw',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'wad', type: 'uint256' }],
    outputs: []
  },
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }]
  }
] as const;

async function wrapETH(
  walletClient: any,
  amount: bigint
) {
  console.log(`Wrapping ${formatUnits(amount, 18)} ETH...`);
  
  const wrapTx = await walletClient.writeContract({
    address: WETH,
    abi: wethAbi,
    functionName: 'deposit',
    value: amount
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: wrapTx });
  console.log('ETH wrapped to WETH:', wrapTx);
  
  return wrapTx;
}

async function unwrapWETH(
  walletClient: any,
  amount: bigint
) {
  console.log(`Unwrapping ${formatUnits(amount, 18)} WETH...`);
  
  const unwrapTx = await walletClient.writeContract({
    address: WETH,
    abi: wethAbi,
    functionName: 'withdraw',
    args: [amount]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: unwrapTx });
  console.log('WETH unwrapped to ETH:', unwrapTx);
  
  return unwrapTx;
}

async function swapETHForToken(
  walletClient: any,
  tokenOut: `0x${string}`,
  ethAmount: bigint,
  slippageBps: number = 50
) {
  // Wrap ETH to WETH first
  await wrapETH(walletClient, ethAmount);
  
  // Then swap WETH for desired token
  return await executeSwap(
    walletClient,
    WETH,
    tokenOut,
    ethAmount,
    slippageBps
  );
}
```

## Aggregator Integration

Use DEX aggregators for best prices:

```typescript
interface QuoteResponse {
  dstAmount: string;
  tx: {
    from: string;
    to: string;
    data: string;
    value: string;
    gas: string;
  };
}

async function get1inchQuote(
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  amount: bigint,
  fromAddress: `0x${string}`,
  slippageBps: number = 50
): Promise<QuoteResponse> {
  const chainId = 8453; // Base
  const slippagePercent = slippageBps / 100;
  
  const url = `https://api.1inch.dev/swap/v5.2/${chainId}/swap?` +
    `src=${tokenIn}&` +
    `dst=${tokenOut}&` +
    `amount=${amount.toString()}&` +
    `from=${fromAddress}&` +
    `slippage=${slippagePercent}&` +
    `disableEstimate=true`;
  
  const response = await fetch(url, {
    headers: {
      'Authorization': `Bearer ${process.env.ONEINCH_API_KEY}`
    }
  });
  
  if (!response.ok) {
    throw new Error(`1inch API error: ${response.statusText}`);
  }
  
  return await response.json();
}

async function executeAggregatorSwap(
  walletClient: any,
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  amountIn: bigint,
  slippageBps: number = 50
) {
  const account = walletClient.account;
  
  // Get quote from aggregator
  const quote = await get1inchQuote(
    tokenIn,
    tokenOut,
    amountIn,
    account.address,
    slippageBps
  );
  
  console.log(`1inch quote: ${quote.dstAmount} tokens out`);
  
  // Approve if needed
  await approveIfNeeded(walletClient, tokenIn, quote.tx.to as `0x${string}`, amountIn);
  
  // Execute swap using aggregator's calldata
  const swapTx = await walletClient.sendTransaction({
    to: quote.tx.to as `0x${string}`,
    data: quote.tx.data as `0x${string}`,
    value: BigInt(quote.tx.value),
    gas: BigInt(quote.tx.gas)
  });
  
  console.log('Aggregator swap executed:', swapTx);
  return swapTx;
}
```

## Gas Optimization

Optimize gas costs for DeFi operations:

```typescript
async function batchApprovals(
  walletClient: any,
  tokens: `0x${string}`[],
  spenders: `0x${string}`[]
) {
  if (tokens.length !== spenders.length) {
    throw new Error('Tokens and spenders arrays must be same length');
  }
  
  const calls = tokens.map((token, i) => ({
    target: token,
    allowFailure: false,
    callData: encodeFunctionData({
      abi: erc20Abi,
      functionName: 'approve',
      args: [spenders[i], maxUint256]
    })
  }));
  
  const multicallTx = await walletClient.writeContract({
    address: MULTICALL3,
    abi: multicallAbi,
    functionName: 'aggregate3',
    args: [calls]
  });
  
  console.log('Batch approvals executed:', multicallTx);
  return multicallTx;
}

async function estimateOptimalGasPrice() {
  const block = await publicClient.getBlock();
  const baseFee = block.baseFeePerGas;
  
  if (!baseFee) {
    return { maxFeePerGas: parseUnits('0.01', 9), maxPriorityFeePerGas: parseUnits('0.001', 9) };
  }
  
  // Base: 2x base fee + 0.001 gwei priority
  const maxFeePerGas = baseFee * BigInt(2);
  const maxPriorityFeePerGas = parseUnits('0.001', 9);
  
  return { maxFeePerGas, maxPriorityFeePerGas };
}
```

## Error Handling

Robust error handling for DeFi operations:

```typescript
class SwapError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: any
  ) {
    super(message);
    this.name = 'SwapError';
  }
}

async function safeSwap(
  walletClient: any,
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  amountIn: bigint,
  slippageBps: number = 50
) {
  try {
    // Validate inputs
    if (amountIn <= BigInt(0)) {
      throw new SwapError('Invalid amount', 'INVALID_AMOUNT');
    }
    
    // Check balance
    const balance = await publicClient.readContract({
      address: tokenIn,
      abi: erc20Abi,
      functionName: 'balanceOf',
      args: [walletClient.account.address]
    });
    
    if (balance < amountIn) {
      throw new SwapError(
        'Insufficient balance',
        'INSUFFICIENT_BALANCE',
        { balance, required: amountIn }
      );
    }
    
    // Execute swap
    return await executeSwap(walletClient, tokenIn, tokenOut, amountIn, slippageBps);
    
  } catch (error: any) {
    if (error instanceof SwapError) {
      throw error;
    }
    
    // Parse common revert reasons
    if (error.message.includes('insufficient output amount')) {
      throw new SwapError(
        'Slippage exceeded, increase slippage tolerance',
        'SLIPPAGE_EXCEEDED'
      );
    }
    
    if (error.message.includes('transfer amount exceeds allowance')) {
      throw new SwapError(
        'Insufficient approval, re-approve token',
        'INSUFFICIENT_APPROVAL'
      );
    }
    
    if (error.message.includes('expired')) {
      throw new SwapError(
        'Transaction deadline expired',
        'DEADLINE_EXPIRED'
      );
    }
    
    // Unknown error
    throw new SwapError(
      `Swap failed: ${error.message}`,
      'UNKNOWN_ERROR',
      error
    );
  }
}
```

## Common Patterns

**Approve Once Pattern:** Approve max uint256 instead of exact amounts to save gas on future swaps.

**Quote Before Execute:** Always get a fresh quote immediately before swap execution to account for price movement.

**Deadline Buffer:** Set deadline 10-20 minutes in future, not too short (fails on congestion) or too long (stale quotes).

**Slippage Tiers:** Use 0.1-0.5% for stablecoins, 0.5-1% for blue chips, 1-3% for volatile assets.

**Multi-hop Optimization:** Compare direct routes vs multi-hop routes for best effective price after gas costs.

**Batch Operations:** Use multicall to batch approvals, swaps, or other operations atomically.

## Gotchas

**Approval Front-Running:** Use max approval to avoid needing multiple approval transactions. Some older tokens have approve race condition.

**Quote Staleness:** Quotes are only valid for current block. Significant price movement can occur between quote and execution.

**MEV Exposure:** Large swaps are visible in mempool and may be front-run. Consider private transaction services.

**Gas Estimation:** DEX operations can be gas-heavy. Always estimate gas and add 20% buffer.

**Failed Transactions:** Transaction can fail after approval succeeds. Always check both transaction statuses.

**Decimal Precision:** Different tokens have different decimals (USDC: 6, WETH: 18). Be careful with amount conversions.

**Native ETH:** Most DEXes don't accept native ETH directly. Must wrap to WETH first.

**Router Updates:** DEX routers can be upgraded. Always verify current router address before operations.

**Slippage vs Price Impact:** Slippage protection protects against unfavorable execution. Large trades also have inherent price impact from moving the pool ratio.