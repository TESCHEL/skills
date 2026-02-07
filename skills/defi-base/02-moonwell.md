# Moonwell Lending on Base

## Overview

Moonwell is a fork of Compound v2 deployed on Base, providing lending and borrowing markets for major assets. Key features:

- Isolated lending pools for different assets
- Algorithmic interest rate models based on utilization
- Over-collateralized borrowing with collateral factors
- Liquidation mechanism to maintain solvency
- Governance through WELL tokens

Moonwell USDC Market (mUSDC): 0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22

This lesson covers supplying assets, borrowing against collateral, interest calculations, and managing positions on Moonwell.

## Market Interface

Moonwell markets (mTokens) implement the Compound v2 CErc20 interface:

```typescript
import { createPublicClient, createWalletClient, http, parseUnits, formatUnits } from 'viem';
import { base } from 'viem/chains';

const MOONWELL_USDC = '0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22';
const USDC = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const MOONWELL_COMPTROLLER = '0xfBb21d0380beE3312B33c4353c8936a0F13EF26C';

const mTokenAbi = [
  {
    name: 'mint',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'mintAmount', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'redeem',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'redeemTokens', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'redeemUnderlying',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'redeemAmount', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'borrow',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'borrowAmount', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'repayBorrow',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'repayAmount', type: 'uint256' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'borrowBalanceCurrent',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'balanceOfUnderlying',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'owner', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'exchangeRateCurrent',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'supplyRatePerBlock',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'borrowRatePerBlock',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'totalBorrows',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'totalSupply',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'getCash',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }]
  }
] as const;

const comptrollerAbi = [
  {
    name: 'enterMarkets',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'mTokens', type: 'address[]' }],
    outputs: [{ name: '', type: 'uint256[]' }]
  },
  {
    name: 'exitMarket',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'mToken', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }]
  },
  {
    name: 'getAccountLiquidity',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [
      { name: '', type: 'uint256' },
      { name: 'liquidity', type: 'uint256' },
      { name: 'shortfall', type: 'uint256' }
    ]
  },
  {
    name: 'markets',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'mToken', type: 'address' }],
    outputs: [
      { name: 'isListed', type: 'bool' },
      { name: 'collateralFactorMantissa', type: 'uint256' },
      { name: 'isComped', type: 'bool' }
    ]
  }
] as const;

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});
```

## Supplying Assets

Supply USDC to Moonwell to earn interest:

```typescript
async function supplyUSDC(
  walletClient: any,
  amount: bigint
) {
  const account = walletClient.account;
  
  // Check USDC balance
  const erc20Abi = [
    {
      name: 'balanceOf',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'account', type: 'address' }],
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
  
  const balance = await publicClient.readContract({
    address: USDC,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [account.address]
  });
  
  console.log(`USDC Balance: ${formatUnits(balance, 6)}`);
  
  if (balance < amount) {
    throw new Error('Insufficient USDC balance');
  }
  
  // Approve USDC to mUSDC market
  console.log('Approving USDC...');
  const approveTx = await walletClient.writeContract({
    address: USDC,
    abi: erc20Abi,
    functionName: 'approve',
    args: [MOONWELL_USDC, amount]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveTx });
  
  // Supply (mint mTokens)
  console.log(`Supplying ${formatUnits(amount, 6)} USDC...`);
  const mintTx = await walletClient.writeContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'mint',
    args: [amount]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: mintTx });
  console.log('Supply successful:', mintTx);
  
  // Get exchange rate to calculate mUSDC received
  const exchangeRate = await publicClient.readContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'exchangeRateCurrent'
  });
  
  const mTokensReceived = (amount * BigInt(1e18)) / exchangeRate;
  console.log(`Received ${formatUnits(mTokensReceived, 8)} mUSDC`);
  console.log(`Exchange rate: ${formatUnits(exchangeRate, 18)}`);
  
  return { hash: mintTx, mTokensReceived, receipt };
}
```

## Checking Supply APY

Calculate current supply APY from rate per block:

```typescript
async function getSupplyAPY(mToken: `0x${string}`) {
  // Get supply rate per block (in mantissa format, 1e18)
  const supplyRatePerBlock = await publicClient.readContract({
    address: mToken,
    abi: mTokenAbi,
    functionName: 'supplyRatePerBlock'
  });
  
  // Base L2 produces blocks every 2 seconds
  const blocksPerYear = (365 * 24 * 60 * 60) / 2; // ~15,768,000 blocks
  
  // APY = (1 + ratePerBlock) ^ blocksPerYear - 1
  // Simplified approximation: APY = ratePerBlock * blocksPerYear
  const supplyAPY = Number(supplyRatePerBlock * BigInt(blocksPerYear)) / 1e18;
  
  console.log(`Supply Rate Per Block: ${formatUnits(supplyRatePerBlock, 18)}`);
  console.log(`Supply APY: ${(supplyAPY * 100).toFixed(2)}%`);
  
  return supplyAPY;
}
```

## Entering Markets as Collateral

Enable supplied assets as collateral for borrowing:

```typescript
async function enterMarket(
  walletClient: any,
  mToken: `0x${string}`
) {
  // Check collateral factor
  const marketInfo = await publicClient.readContract({
    address: MOONWELL_COMPTROLLER,
    abi: comptrollerAbi,
    functionName: 'markets',
    args: [mToken]
  });
  
  const collateralFactor = Number(marketInfo[1]) / 1e18;
  console.log(`Collateral Factor: ${(collateralFactor * 100).toFixed(2)}%`);
  
  // Enter market
  const enterTx = await walletClient.writeContract({
    address: MOONWELL_COMPTROLLER,
    abi: comptrollerAbi,
    functionName: 'enterMarkets',
    args: [[mToken]]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: enterTx });
  console.log('Entered market:', enterTx);
  
  return { hash: enterTx, collateralFactor, receipt };
}
```

## Borrowing Assets

Borrow against supplied collateral:

```typescript
async function borrowUSDC(
  walletClient: any,
  borrowAmount: bigint
) {
  const account = walletClient.account;
  
  // Check account liquidity
  const liquidity = await publicClient.readContract({
    address: MOONWELL_COMPTROLLER,
    abi: comptrollerAbi,
    functionName: 'getAccountLiquidity',
    args: [account.address]
  });
  
  const availableLiquidity = liquidity[1];
  const shortfall = liquidity[2];
  
  console.log(`Available Liquidity: ${formatUnits(availableLiquidity, 18)} USD`);
  console.log(`Shortfall: ${formatUnits(shortfall, 18)} USD`);
  
  if (shortfall > BigInt(0)) {
    throw new Error('Account has shortfall, cannot borrow');
  }
  
  // Convert borrow amount to USD value (assuming USDC = $1)
  const borrowValueUSD = borrowAmount; // USDC has 6 decimals, scale to 18
  const borrowValueScaled = borrowValueUSD * BigInt(1e12); // Scale to 18 decimals
  
  if (borrowValueScaled > availableLiquidity) {
    throw new Error(`Insufficient liquidity. Available: ${formatUnits(availableLiquidity, 18)} USD`);
  }
  
  // Execute borrow
  console.log(`Borrowing ${formatUnits(borrowAmount, 6)} USDC...`);
  const borrowTx = await walletClient.writeContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'borrow',
    args: [borrowAmount]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: borrowTx });
  console.log('Borrow successful:', borrowTx);
  
  // Check new borrow balance
  const borrowBalance = await publicClient.readContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'borrowBalanceCurrent',
    args: [account.address]
  });
  
  console.log(`Total Borrow Balance: ${formatUnits(borrowBalance, 6)} USDC`);
  
  return { hash: borrowTx, borrowBalance, receipt };
}
```

## Repaying Borrowed Assets

Repay borrowed USDC to reduce debt:

```typescript
async function repayUSDC(
  walletClient: any,
  repayAmount: bigint // Use max uint256 to repay all
) {
  const account = walletClient.account;
  
  // Get current borrow balance
  const borrowBalance = await publicClient.readContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'borrowBalanceCurrent',
    args: [account.address]
  });
  
  console.log(`Current Borrow: ${formatUnits(borrowBalance, 6)} USDC`);
  
  if (borrowBalance === BigInt(0)) {
    console.log('No outstanding borrows');
    return;
  }
  
  // Determine actual repay amount
  const actualRepayAmount = repayAmount > borrowBalance ? borrowBalance : repayAmount;
  
  // Approve USDC
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
  
  console.log('Approving USDC for repayment...');
  const approveTx = await walletClient.writeContract({
    address: USDC,
    abi: erc20Abi,
    functionName: 'approve',
    args: [MOONWELL_USDC, actualRepayAmount]
  });
  await publicClient.waitForTransactionReceipt({ hash: approveTx });
  
  // Repay borrow
  console.log(`Repaying ${formatUnits(actualRepayAmount, 6)} USDC...`);
  const repayTx = await walletClient.writeContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'repayBorrow',
    args: [actualRepayAmount]
  });
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: repayTx });
  console.log('Repayment successful:', repayTx);
  
  // Check remaining borrow balance
  const newBorrowBalance = await publicClient.readContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'borrowBalanceCurrent',
    args: [account.address]
  });
  
  console.log(`Remaining Borrow: ${formatUnits(newBorrowBalance, 6)} USDC`);
  
  return { hash: repayTx, newBorrowBalance, receipt };
}
```

## Withdrawing Supplied Assets

Withdraw supplied assets by redeeming mTokens:

```typescript
async function withdrawUSDC(
  walletClient: any,
  amount: bigint, // Amount of underlying USDC to withdraw
  redeemType: 'underlying' | 'mTokens' = 'underlying'
) {
  const account = walletClient.account;
  
  // Check if withdrawal would cause shortfall
  const liquidity = await publicClient.readContract({
    address: MOONWELL_COMPTROLLER,
    abi: comptrollerAbi,
    functionName: 'getAccountLiquidity',
    args: [account.address]
  });
  
  console.log(`Available Liquidity: ${formatUnits(liquidity[1], 18)} USD`);
  
  let redeemTx: `0x${string}`;
  
  if (redeemType === 'underlying') {
    // Redeem specific amount of underlying
    console.log(`Withdrawing ${formatUnits(amount, 6)} USDC...`);
    redeemTx = await walletClient.writeContract({
      address: MOONWELL_USDC,
      abi: mTokenAbi,
      functionName: 'redeemUnderlying',
      args: [amount]
    });
  } else {
    // Redeem specific amount of mTokens
    console.log(`Redeeming ${formatUnits(amount, 8)} mUSDC...`);
    redeemTx = await walletClient.writeContract({
      address: MOONWELL_USDC,
      abi: mTokenAbi,
      functionName: 'redeem',
      args: [amount]
    });
  }
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash: redeemTx });
  console.log('Withdrawal successful:', redeemTx);
  
  return { hash: redeemTx, receipt };
}
```

## Health Factor Calculation

Monitor account health to avoid liquidation:

```typescript
async function checkAccountHealth(account: `0x${string}`) {
  // Get account liquidity
  const liquidity = await publicClient.readContract({
    address: MOONWELL_COMPTROLLER,
    abi: comptrollerAbi,
    functionName: 'getAccountLiquidity',
    args: [account]
  });
  
  const availableLiquidity = liquidity[1];
  const shortfall = liquidity[2];
  
  // Get supplied balance
  const supplyBalance = await publicClient.readContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'balanceOfUnderlying',
    args: [account]
  });
  
  // Get borrow balance
  const borrowBalance = await publicClient.readContract({
    address: MOONWELL_USDC,
    abi: mTokenAbi,
    functionName: 'borrowBalanceCurrent',
    args: [account]
  });
  
  // Get collateral factor
  const marketInfo = await publicClient.readContract({
    address: MOONWELL_COMPTROLLER,
    abi: comptrollerAbi,
    functionName: 'markets',
    args: [MOONWELL_USDC]
  });
  
  const collateralFactor = Number(marketInfo[1]) / 1e18;
  
  console.log('Account Health:');
  console.log(`Supplied: ${formatUnits(supplyBalance, 6)} USDC`);
  console.log(`Borrowed: ${formatUnits(borrowBalance, 6)} USDC`);
  console.log(`Collateral Factor: ${(collateralFactor * 100).toFixed(2)}%`);
  console.log(`Available Liquidity: ${formatUnits(availableLiquidity, 18)} USD`);
  console.log(`Shortfall: ${formatUnits(shortfall, 18)} USD`);
  
  // Calculate health factor (borrowCapacity / borrowed)
  const borrowCapacity = Number(supplyBalance) * collateralFactor;
  const healthFactor = borrowCapacity / Number(borrowBalance);
  
  console.log(`Health Factor: ${healthFactor.toFixed(2)}`);
  
  if (healthFactor < 1.1) {
    console.warn('WARNING: Account at risk of liquidation!');
  } else if (healthFactor < 1.5) {
    console.warn('CAUTION: Low health factor');
  }
  
  return {
    supplyBalance,
    borrowBalance,
    collateralFactor,
    availableLiquidity,
    shortfall,
    healthFactor
  };
}
```

## Common Patterns

**Supply -> Enter Market -> Borrow Flow:**
```typescript
async function supplyAndBorrow(
  walletClient: any,
  supplyAmount: bigint,
  borrowAmount: bigint
) {
  // 1. Supply collateral
  await supplyUSDC(walletClient, supplyAmount);
  
  // 2. Enter market to use as collateral
  await enterMarket(walletClient, MOONWELL_USDC);
  
  // 3. Borrow against collateral
  await borrowUSDC(walletClient, borrowAmount);
  
  // 4. Check final health
  await checkAccountHealth(walletClient.account.address);
}
```

**Interest Accrual:** Interest accrues every block. Call `exchangeRateCurrent()` or `borrowBalanceCurrent()` to update state before reads.

**Utilization Rate:** Borrow APY increases as utilization increases. Monitor `totalBorrows / (totalSupply * exchangeRate + totalBorrows)` for utilization.

**Liquidation Threshold:** Accounts become liquidatable when borrow value exceeds collateral value * collateral factor. Maintain health factor > 1.2 for safety.

## Gotchas

**Exchange Rate Updates:** The exchange rate increases as interest accrues. Always call non-view function (`exchangeRateCurrent`) before calculating mToken amounts, not the view function (`exchangeRateStored`).

**Borrow Balance Accrual:** Similar to exchange rate, borrow balances grow with interest. Use `borrowBalanceCurrent()` not `borrowBalanceStored()`.

**Repay Max Amount:** To repay entire borrow, pass max uint256 (`2^256 - 1`). Don't try to calculate exact amount due to per-block interest accrual.

**Decimal Precision:** USDC has 6 decimals, but mUSDC has 8 decimals. Exchange rate has 18 decimals. Be careful with conversions.

**Enter Markets Required:** You must call `enterMarkets()` on Comptroller before supplied assets can be used as collateral for borrowing.

**Exit Market Restrictions:** Cannot exit market if borrowed assets depend on it as collateral. Must repay borrows first.

**Liquidation Timing:** Liquidations can happen suddenly during price volatility. Monitor health factor closely and maintain buffer.

**Gas Costs:** Moonwell operations can be gas-intensive due to interest accrual calculations. Budget sufficient gas for transactions.