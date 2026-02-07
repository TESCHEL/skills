# Aerodrome DEX on Base

## Overview

Aerodrome is the leading decentralized exchange on Base, using the ve(3,3) model pioneered by Solidly. It features:

- Volatile pools for uncorrelated asset pairs (constant product AMM)
- Stable pools for like-kind assets (stable swap curve)
- Vote-escrowed tokenomics (veAERO) for governance and fee distribution
- Optimized routing across multiple pool types
- Deep liquidity incentivized through gauge voting

Router contract: 0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43

This lesson covers swapping tokens, providing liquidity, and understanding pool mechanics on Aerodrome.

## Router Interface

The Aerodrome Router provides the primary interface for swaps and liquidity operations:

```typescript
import { createPublicClient, createWalletClient, http, parseUnits, formatUnits } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const AERODROME_ROUTER = '0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43';
const USDC = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const WETH = '0x4200000000000000000000000000000000000006';

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
  },
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
    name: 'addLiquidity',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'tokenA', type: 'address' },
      { name: 'tokenB', type: 'address' },
      { name: 'stable', type: 'bool' },
      { name: 'amountADesired', type: 'uint256' },
      { name: 'amountBDesired', type: 'uint256' },
      { name: 'amountAMin', type: 'uint256' },
      { name: 'amountBMin', type: 'uint256' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' }
    ],
    outputs: [
      { name: 'amountA', type: 'uint256' },
      { name: 'amountB', type: 'uint256' },
      { name: 'liquidity', type: 'uint256' }
    ]
  },
  {
    name: 'removeLiquidity',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'tokenA', type: 'address' },
      { name: 'tokenB', type: 'address' },
      { name: 'stable', type: 'bool' },
      { name: 'liquidity', type: 'uint256' },
      { name: 'amountAMin', type: 'uint256' },
      { name: 'amountBMin', type: 'uint256' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' }
    ],
    outputs: [
      { name: 'amountA', type: 'uint256' },
      { name: 'amountB', type: 'uint256' }
    ]
  }
] as const;

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});
```

## Swapping Tokens

Basic token swap with slippage protection:

```typescript
const AERODROME_FACTORY = '0x420DD381b31aEf6683db6B902084cB0FFECe40Da';

async function swapUSDCForWETH(
  walletClient: any,
  amountIn: bigint,
  slippageBps: number = 50 // 0.5%
) {
  const account = walletClient.account;
  
  // Define route: USDC -> WETH (volatile pool)
  const routes = [{
    from: USDC,
    to: WETH,
    stable: false, // volatile pool for uncorrelated assets
    factory: AERODROME_FACTORY
  }];
  
  // Get quote
  const amounts = await publicClient.readContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'getAmountsOut',
    args: [amountIn, routes]
  });
  
  const amountOut = amounts[1];
  const amountOutMin = (amountOut * BigInt(10000 - slippageBps)) / BigInt(10000);
  
  console.log(`Quote: ${formatUnits(amountIn, 6)} USDC -> ${formatUnits(amountOut, 18)} WETH`);
  console.log(`Min output: ${formatUnits(amountOutMin, 18)} WETH (${slippageBps / 100}% slippage)`);
  
  // Check USDC approval
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
  
  const allowance = await publicClient.readContract({
    address: USDC,
    abi: erc20Abi,
    functionName: 'allowance',
    args: [account.address, AERODROME_ROUTER]
  });
  
  if (allowance < amountIn) {
    console.log('Approving USDC...');
    const approveTx = await walletClient.writeContract({
      address: USDC,
      abi: erc20Abi,
      functionName: 'approve',
      args: [AERODROME_ROUTER, amountIn]
    });
    await publicClient.waitForTransactionReceipt({ hash: approveTx });
    console.log('Approved:', approveTx);
  }
  
  // Execute swap
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 1200); // 20 minutes
  
  const swapTx = await walletClient.writeContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'swapExactTokensForTokens',
    args: [
      amountIn,
      amountOutMin,
      routes,
      account.address,
      deadline
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: swapTx });
  console.log('Swap executed:', swapTx);
  console.log('Status:', receipt.status);
  
  return { hash: swapTx, amountOut, receipt };
}
```

## Multi-Hop Routing

Route through multiple pools for better prices:

```typescript
async function swapWithMultipleHops(
  walletClient: any,
  tokenIn: `0x${string}`,
  tokenOut: `0x${string}`,
  amountIn: bigint,
  intermediateToken: `0x${string}`
) {
  // Example: USDC -> WETH -> DAI
  // Route 1: USDC -> WETH (volatile)
  // Route 2: WETH -> DAI (volatile)
  const routes = [
    {
      from: tokenIn,
      to: intermediateToken,
      stable: false,
      factory: AERODROME_FACTORY
    },
    {
      from: intermediateToken,
      to: tokenOut,
      stable: false,
      factory: AERODROME_FACTORY
    }
  ];
  
  // Get quote for multi-hop route
  const amounts = await publicClient.readContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'getAmountsOut',
    args: [amountIn, routes]
  });
  
  console.log('Route amounts:', amounts.map(a => a.toString()));
  const finalAmount = amounts[amounts.length - 1];
  const amountOutMin = (finalAmount * BigInt(9950)) / BigInt(10000); // 0.5% slippage
  
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 1200);
  
  const swapTx = await walletClient.writeContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'swapExactTokensForTokens',
    args: [amountIn, amountOutMin, routes, walletClient.account.address, deadline]
  });
  
  return { hash: swapTx, finalAmount };
}
```

## Providing Liquidity

Add liquidity to pools and receive LP tokens:

```typescript
async function addLiquidityToPool(
  walletClient: any,
  tokenA: `0x${string}`,
  tokenB: `0x${string}`,
  stable: boolean,
  amountA: bigint,
  amountB: bigint,
  slippageBps: number = 100 // 1%
) {
  const account = walletClient.account;
  
  // Approve both tokens
  const erc20Abi = [
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
  
  console.log('Approving tokens...');
  const approveA = await walletClient.writeContract({
    address: tokenA,
    abi: erc20Abi,
    functionName: 'approve',
    args: [AERODROME_ROUTER, amountA]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveA });
  
  const approveB = await walletClient.writeContract({
    address: tokenB,
    abi: erc20Abi,
    functionName: 'approve',
    args: [AERODROME_ROUTER, amountB]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveB });
  
  // Calculate minimum amounts with slippage
  const amountAMin = (amountA * BigInt(10000 - slippageBps)) / BigInt(10000);
  const amountBMin = (amountB * BigInt(10000 - slippageBps)) / BigInt(10000);
  
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 1200);
  
  console.log(`Adding liquidity: ${amountA} tokenA + ${amountB} tokenB`);
  console.log(`Stable pool: ${stable}`);
  
  const addLiqTx = await walletClient.writeContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'addLiquidity',
    args: [
      tokenA,
      tokenB,
      stable,
      amountA,
      amountB,
      amountAMin,
      amountBMin,
      account.address,
      deadline
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: addLiqTx });
  
  // Parse event logs to get liquidity amount
  console.log('Liquidity added:', addLiqTx);
  console.log('Receipt:', receipt.status);
  
  return { hash: addLiqTx, receipt };
}
```

## Removing Liquidity

Withdraw from pools by burning LP tokens:

```typescript
async function removeLiquidityFromPool(
  walletClient: any,
  tokenA: `0x${string}`,
  tokenB: `0x${string}`,
  stable: boolean,
  liquidity: bigint,
  slippageBps: number = 100
) {
  // First, need to find the pair address and approve LP tokens
  const pairFactoryAbi = [
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
  
  const pairAddress = await publicClient.readContract({
    address: AERODROME_FACTORY,
    abi: pairFactoryAbi,
    functionName: 'getPair',
    args: [tokenA, tokenB, stable]
  });
  
  console.log('Pair address:', pairAddress);
  
  // Approve LP tokens
  const erc20Abi = [
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
  
  const approveLp = await walletClient.writeContract({
    address: pairAddress,
    abi: erc20Abi,
    functionName: 'approve',
    args: [AERODROME_ROUTER, liquidity]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveLp });
  
  // Remove liquidity with 1% slippage on both tokens
  const deadline = BigInt(Math.floor(Date.now() / 1000) + 1200);
  
  const removeLiqTx = await walletClient.writeContract({
    address: AERODROME_ROUTER,
    abi: routerAbi,
    functionName: 'removeLiquidity',
    args: [
      tokenA,
      tokenB,
      stable,
      liquidity,
      BigInt(0), // amountAMin (calculate properly in production)
      BigInt(0), // amountBMin (calculate properly in production)
      walletClient.account.address,
      deadline
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: removeLiqTx });
  console.log('Liquidity removed:', removeLiqTx);
  
  return { hash: removeLiqTx, receipt };
}
```

## Common Patterns

**Pool Type Selection:**
- Use `stable: false` (volatile pools) for uncorrelated assets (USDC/WETH, WETH/AERO)
- Use `stable: true` (stable pools) for like-kind assets (USDC/DAI, USDC/USDT)
- Stable pools use Curve-style invariant, better for low-slippage pegged asset swaps
- Volatile pools use constant product (x * y = k)

**Deadline Management:**
- Always set reasonable deadline (typically 10-20 minutes from now)
- Convert to Unix timestamp: `Math.floor(Date.now() / 1000) + 1200`
- Transactions revert if executed after deadline

**Slippage Protection:**
- Calculate `amountOutMin` as percentage of expected output
- Typical slippage: 0.5% for stable pairs, 1-2% for volatile
- Large trades need higher slippage tolerance
- Price impact increases non-linearly with trade size

**Route Optimization:**
- Check both direct routes and multi-hop routes
- Compare quotes before execution
- Consider gas costs for multi-hop routes
- Some routes may be better even with extra hops

## Gotchas

**Factory Address:** Aerodrome uses its own factory contract. Always use 0x420DD381b31aEf6683db6B902084cB0FFECe40Da in route structs.

**Stable vs Volatile:** Using wrong pool type causes high slippage or revert. Stable pools only work well for pegged assets.

**Approval Race Conditions:** Some tokens (not USDC) require approval to be 0 before changing. Set to 0 first, then set to desired amount.

**LP Token Accounting:** LP tokens are ERC-20 tokens. Track them separately. The pair address is the LP token address.

**Imbalanced Liquidity:** When adding liquidity, actual amounts used may differ from input amounts based on pool ratios. Check return values.

**Quote Staleness:** Quotes from `getAmountsOut` are only valid for current block. Prices change between quote and execution. Always use slippage protection.

**Deadline Too Short:** Setting deadline too close (< 5 minutes) may cause reverts during network congestion. Use 10-20 minutes for safety.

**WETH Wrapping:** Aerodrome router doesn't automatically wrap ETH. Use WETH address directly and wrap ETH separately if needed.