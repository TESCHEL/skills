# Morpho Blue on Base

## Overview

Morpho Blue is a trustless lending primitive that enables permissionless creation of isolated lending markets. Key characteristics:

- Isolated risk: Each market is independent with its own parameters
- Permissionless: Anyone can create markets with custom parameters
- Minimal: Core protocol is immutable and non-upgradeable
- Efficient: Direct peer-to-peer matching reduces intermediation
- Oracle-agnostic: Markets can use any oracle implementation

Morpho Blue on Base: 0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb

This lesson covers creating markets, supplying and borrowing, managing positions, and using MetaMorpho vaults.

## Core Concepts

**Market Parameters:**
- Loan Token: Asset being lent
- Collateral Token: Asset used as collateral
- Oracle: Price feed for collateralization ratio
- IRM (Interest Rate Model): Determines borrow rates
- LLTV (Liquidation Loan-to-Value): Maximum borrow ratio before liquidation

**Market ID:** Unique identifier computed from parameters hash. Each parameter combination creates a new isolated market.

**MetaMorpho Vaults:** Curated lending vaults that deposit into multiple Morpho markets with risk management.

## Morpho Blue Interface

```typescript
import { createPublicClient, createWalletClient, http, parseUnits, formatUnits, keccak256, encodePacked } from 'viem';
import { base } from 'viem/chains';

const MORPHO_BLUE = '0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb';
const USDC = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const WETH = '0x4200000000000000000000000000000000000006';

const morphoAbi = [
  {
    name: 'supply',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'marketParams',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      },
      { name: 'assets', type: 'uint256' },
      { name: 'shares', type: 'uint256' },
      { name: 'onBehalf', type: 'address' },
      { name: 'data', type: 'bytes' }
    ],
    outputs: [
      { name: 'assetsSupplied', type: 'uint256' },
      { name: 'sharesSupplied', type: 'uint256' }
    ]
  },
  {
    name: 'withdraw',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'marketParams',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      },
      { name: 'assets', type: 'uint256' },
      { name: 'shares', type: 'uint256' },
      { name: 'onBehalf', type: 'address' },
      { name: 'receiver', type: 'address' }
    ],
    outputs: [
      { name: 'assetsWithdrawn', type: 'uint256' },
      { name: 'sharesWithdrawn', type: 'uint256' }
    ]
  },
  {
    name: 'borrow',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'marketParams',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      },
      { name: 'assets', type: 'uint256' },
      { name: 'shares', type: 'uint256' },
      { name: 'onBehalf', type: 'address' },
      { name: 'receiver', type: 'address' }
    ],
    outputs: [
      { name: 'assetsBorrowed', type: 'uint256' },
      { name: 'sharesBorrowed', type: 'uint256' }
    ]
  },
  {
    name: 'repay',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'marketParams',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      },
      { name: 'assets', type: 'uint256' },
      { name: 'shares', type: 'uint256' },
      { name: 'onBehalf', type: 'address' },
      { name: 'data', type: 'bytes' }
    ],
    outputs: [
      { name: 'assetsRepaid', type: 'uint256' },
      { name: 'sharesRepaid', type: 'uint256' }
    ]
  },
  {
    name: 'supplyCollateral',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'marketParams',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      },
      { name: 'assets', type: 'uint256' },
      { name: 'onBehalf', type: 'address' },
      { name: 'data', type: 'bytes' }
    ],
    outputs: []
  },
  {
    name: 'withdrawCollateral',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'marketParams',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      },
      { name: 'assets', type: 'uint256' },
      { name: 'onBehalf', type: 'address' },
      { name: 'receiver', type: 'address' }
    ],
    outputs: []
  },
  {
    name: 'market',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'id', type: 'bytes32' }],
    outputs: [
      { name: 'totalSupplyAssets', type: 'uint128' },
      { name: 'totalSupplyShares', type: 'uint128' },
      { name: 'totalBorrowAssets', type: 'uint128' },
      { name: 'totalBorrowShares', type: 'uint128' },
      { name: 'lastUpdate', type: 'uint128' },
      { name: 'fee', type: 'uint128' }
    ]
  },
  {
    name: 'position',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'id', type: 'bytes32' },
      { name: 'user', type: 'address' }
    ],
    outputs: [
      { name: 'supplyShares', type: 'uint256' },
      { name: 'borrowShares', type: 'uint128' },
      { name: 'collateral', type: 'uint128' }
    ]
  },
  {
    name: 'idToMarketParams',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'id', type: 'bytes32' }],
    outputs: [
      {
        name: '',
        type: 'tuple',
        components: [
          { name: 'loanToken', type: 'address' },
          { name: 'collateralToken', type: 'address' },
          { name: 'oracle', type: 'address' },
          { name: 'irm', type: 'address' },
          { name: 'lltv', type: 'uint256' }
        ]
      }
    ]
  }
] as const;

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});
```

## Computing Market ID

Market ID is the keccak256 hash of market parameters:

```typescript
interface MarketParams {
  loanToken: `0x${string}`;
  collateralToken: `0x${string}`;
  oracle: `0x${string}`;
  irm: `0x${string}`;
  lltv: bigint;
}

function computeMarketId(params: MarketParams): `0x${string}` {
  // Encode parameters according to Morpho's encoding
  const encoded = encodePacked(
    ['address', 'address', 'address', 'address', 'uint256'],
    [
      params.loanToken,
      params.collateralToken,
      params.oracle,
      params.irm,
      params.lltv
    ]
  );
  
  return keccak256(encoded);
}

// Example: USDC lending market with WETH collateral
const marketParams: MarketParams = {
  loanToken: USDC,
  collateralToken: WETH,
  oracle: '0x0000000000000000000000000000000000000000', // Replace with actual oracle
  irm: '0x0000000000000000000000000000000000000000', // Replace with actual IRM
  lltv: BigInt(860000000000000000) // 86% LLTV
};

const marketId = computeMarketId(marketParams);
console.log('Market ID:', marketId);
```

## Supplying to Markets

Supply loan tokens to earn interest:

```typescript
async function supplyToMorpho(
  walletClient: any,
  marketParams: MarketParams,
  amount: bigint
) {
  const account = walletClient.account;
  
  // Approve loan token
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
  
  console.log('Approving loan token...');
  const approveTx = await walletClient.writeContract({
    address: marketParams.loanToken,
    abi: erc20Abi,
    functionName: 'approve',
    args: [MORPHO_BLUE, amount]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveTx });
  
  // Supply to market
  // Pass amount in assets parameter, 0 in shares
  console.log(`Supplying ${formatUnits(amount, 6)} to market...`);
  const supplyTx = await walletClient.writeContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'supply',
    args: [
      marketParams,
      amount,        // assets to supply
      BigInt(0),     // shares (0 = use assets)
      account.address, // onBehalf
      '0x'          // data
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: supplyTx });
  console.log('Supply successful:', supplyTx);
  
  // Check position
  const marketId = computeMarketId(marketParams);
  const position = await publicClient.readContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'position',
    args: [marketId, account.address]
  });
  
  console.log(`Supply shares: ${position[0]}`);
  
  return { hash: supplyTx, shares: position[0], receipt };
}
```

## Supplying Collateral and Borrowing

Supply collateral then borrow against it:

```typescript
async function supplyCollateralAndBorrow(
  walletClient: any,
  marketParams: MarketParams,
  collateralAmount: bigint,
  borrowAmount: bigint
) {
  const account = walletClient.account;
  
  // Approve collateral token
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
  
  console.log('Approving collateral...');
  const approveTx = await walletClient.writeContract({
    address: marketParams.collateralToken,
    abi: erc20Abi,
    functionName: 'approve',
    args: [MORPHO_BLUE, collateralAmount]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveTx });
  
  // Supply collateral
  console.log(`Supplying ${formatUnits(collateralAmount, 18)} WETH as collateral...`);
  const supplyCollateralTx = await walletClient.writeContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'supplyCollateral',
    args: [
      marketParams,
      collateralAmount,
      account.address,
      '0x'
    ]
  });
  await publicClient.waitForTransactionReceipt({ hash: supplyCollateralTx });
  console.log('Collateral supplied:', supplyCollateralTx);
  
  // Borrow loan tokens
  console.log(`Borrowing ${formatUnits(borrowAmount, 6)} USDC...`);
  const borrowTx = await walletClient.writeContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'borrow',
    args: [
      marketParams,
      borrowAmount,
      BigInt(0),
      account.address,
      account.address
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: borrowTx });
  console.log('Borrow successful:', borrowTx);
  
  // Check final position
  const marketId = computeMarketId(marketParams);
  const position = await publicClient.readContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'position',
    args: [marketId, account.address]
  });
  
  console.log('Position:');
  console.log(`Collateral: ${position[2]}`);
  console.log(`Borrow shares: ${position[1]}`);
  
  return { supplyHash: supplyCollateralTx, borrowHash: borrowTx, position };
}
```

## Repaying Borrowed Assets

Repay borrowed tokens to reduce debt:

```typescript
async function repayBorrow(
  walletClient: any,
  marketParams: MarketParams,
  repayAmount: bigint
) {
  const account = walletClient.account;
  
  // Approve loan token for repayment
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
  
  console.log('Approving loan token for repayment...');
  const approveTx = await walletClient.writeContract({
    address: marketParams.loanToken,
    abi: erc20Abi,
    functionName: 'approve',
    args: [MORPHO_BLUE, repayAmount]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveTx });
  
  // Repay borrow
  console.log(`Repaying ${formatUnits(repayAmount, 6)} USDC...`);
  const repayTx = await walletClient.writeContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'repay',
    args: [
      marketParams,
      repayAmount,
      BigInt(0),
      account.address,
      '0x'
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: repayTx });
  console.log('Repayment successful:', repayTx);
  
  return { hash: repayTx, receipt };
}
```

## Withdrawing Collateral

Withdraw collateral after reducing debt:

```typescript
async function withdrawCollateral(
  walletClient: any,
  marketParams: MarketParams,
  amount: bigint
) {
  const account = walletClient.account;
  
  console.log(`Withdrawing ${formatUnits(amount, 18)} WETH collateral...`);
  const withdrawTx = await walletClient.writeContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'withdrawCollateral',
    args: [
      marketParams,
      amount,
      account.address,
      account.address
    ]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: withdrawTx });
  console.log('Withdrawal successful:', withdrawTx);
  
  return { hash: withdrawTx, receipt };
}
```

## Checking Market State

Query market utilization and rates:

```typescript
async function getMarketState(marketParams: MarketParams) {
  const marketId = computeMarketId(marketParams);
  
  // Get market data
  const market = await publicClient.readContract({
    address: MORPHO_BLUE,
    abi: morphoAbi,
    functionName: 'market',
    args: [marketId]
  });
  
  const totalSupplyAssets = market[0];
  const totalSupplyShares = market[1];
  const totalBorrowAssets = market[2];
  const totalBorrowShares = market[3];
  const lastUpdate = market[4];
  const fee = market[5];
  
  console.log('Market State:');
  console.log(`Total Supply: ${formatUnits(totalSupplyAssets, 6)} USDC`);
  console.log(`Total Borrow: ${formatUnits(totalBorrowAssets, 6)} USDC`);
  
  // Calculate utilization
  const utilization = Number(totalBorrowAssets) / Number(totalSupplyAssets);
  console.log(`Utilization: ${(utilization * 100).toFixed(2)}%`);
  console.log(`Fee: ${fee}`);
  console.log(`Last Update: ${lastUpdate}`);
  
  return {
    totalSupplyAssets,
    totalBorrowAssets,
    utilization,
    fee,
    lastUpdate
  };
}
```

## MetaMorpho Vault Integration

MetaMorpho vaults aggregate deposits across multiple Morpho markets:

```typescript
const metaMorphoAbi = [
  {
    name: 'deposit',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'assets', type: 'uint256' },
      { name: 'receiver', type: 'address' }
    ],
    outputs: [{ name: 'shares', type: 'uint256' }]
  },
  {
    name: 'withdraw',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'assets', type: 'uint256' },
      { name: 'receiver', type: 'address' },
      { name: 'owner', type: 'address' }
    ],
    outputs: [{ name: 'shares', type: 'uint256' }]
  },
  {
    name: 'totalAssets',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'convertToShares',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'assets', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'convertToAssets',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'shares', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  }
] as const;

async function depositToMetaMorpho(
  walletClient: any,
  vaultAddress: `0x${string}`,
  assetAddress: `0x${string}`,
  amount: bigint
) {
  const account = walletClient.account;
  
  // Approve vault
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
  
  const approveTx = await walletClient.writeContract({
    address: assetAddress,
    abi: erc20Abi,
    functionName: 'approve',
    args: [vaultAddress, amount]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveTx });
  
  // Deposit to vault
  const depositTx = await walletClient.writeContract({
    address: vaultAddress,
    abi: metaMorphoAbi,
    functionName: 'deposit',
    args: [amount, account.address]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: depositTx });
  console.log('Deposit to MetaMorpho vault successful:', depositTx);
  
  return { hash: depositTx, receipt };
}
```

## Common Patterns

**Market Discovery:** Query markets from Morpho API or indexer. Market parameters must match exactly to interact with existing markets.

**Shares vs Assets:** Most operations accept either shares or assets. Pass 0 for unused parameter. Shares represent proportional ownership.

**Oracle Validation:** Always verify oracle is returning reasonable prices before large operations. Stale or manipulated oracles cause liquidations.

**LLTV Selection:** Higher LLTV (closer to 100%) means more leverage but higher liquidation risk. Common values: 70-90%.

**Interest Accrual:** Interest accrues continuously. Borrow balance increases every block based on utilization and IRM.

## Gotchas

**Isolated Markets:** Each market is completely isolated. Collateral in one market cannot be used for borrows in another market.

**Market Creation:** Creating markets is permissionless but requires careful parameter selection. Wrong parameters can lead to immediate liquidation risk.

**Oracle Dependencies:** Markets rely entirely on oracles for pricing. Oracle failure or manipulation directly affects liquidation thresholds.

**IRM Variations:** Different IRMs produce different rate curves. Some may have jump rates at high utilization.

**Share Conversion:** Share-to-asset conversion changes as interest accrues. Always check current conversion rate.

**No Native ETH:** Morpho Blue doesn't handle native ETH. Must use WETH for ETH-based markets.

**Liquidation Timing:** Liquidations happen instantly when LTV exceeds LLTV. No grace period.

**Fee Accumulation:** Fees accrue to DAO. Check fee parameter to understand protocol take rate.

**Gas Optimization:** Morpho is gas-efficient but still costs gas. Batch operations when possible.