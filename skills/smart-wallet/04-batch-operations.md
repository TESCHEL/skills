# Batch Operations for Efficient Smart Wallet Execution

## Overview

Batch operations allow multiple contract calls to be executed atomically in a single UserOperation. This is one of the most powerful features of smart wallets, enabling complex multi-step workflows to succeed or fail together.

Benefits of batching:
- Atomic execution (all or nothing)
- Reduced gas costs (one UserOperation overhead instead of multiple)
- Better UX (one approval, one transaction)
- Complex DeFi strategies in single operation
- Simplified error handling

## Batch Execution Methods

Smart wallets typically provide two batch execution methods:

```typescript
const SMART_WALLET_ABI = parseAbi([
  // Single call execution
  'function execute(address dest, uint256 value, bytes calldata func) external',
  
  // Batch call execution
  'function executeBatch(address[] calldata dest, uint256[] calldata value, bytes[] calldata func) external'
]);

interface Call {
  target: Address;
  value: bigint;
  data: Hex;
}
```

## Creating Batch UserOperations

Encode multiple calls into a single UserOperation:

```typescript
import { encodeFunctionData, encodePacked } from 'viem';

async function createBatchUserOp(
  smartWalletAddress: Address,
  calls: Call[],
  ownerAccount: ReturnType<typeof privateKeyToAccount>
): Promise<UserOperation> {
  // Extract targets, values, and data
  const targets = calls.map(c => c.target);
  const values = calls.map(c => c.value);
  const dataArray = calls.map(c => c.data);

  // Encode batch execution
  const callData = encodeFunctionData({
    abi: SMART_WALLET_ABI,
    functionName: 'executeBatch',
    args: [targets, values, dataArray]
  });

  // Get nonce
  const nonce = await publicClient.readContract({
    address: ENTRYPOINT_ADDRESS,
    abi: parseAbi(['function getNonce(address sender, uint192 key) external view returns (uint256 nonce)']),
    functionName: 'getNonce',
    args: [smartWalletAddress, 0n]
  });

  // Check deployment
  const code = await publicClient.getBytecode({ address: smartWalletAddress });
  const initCode = (code && code !== '0x') ? '0x' : getInitCode(ownerAccount.address);

  // Get gas prices
  const feeData = await publicClient.estimateFeesPerGas();

  // Estimate gas for batch (multiply by number of calls + overhead)
  const estimatedGas = BigInt(calls.length) * 100000n + 200000n;

  const userOp: UserOperation = {
    sender: smartWalletAddress,
    nonce,
    initCode,
    callData,
    callGasLimit: estimatedGas,
    verificationGasLimit: 500000n,
    preVerificationGas: 50000n + BigInt(calls.length) * 10000n,
    maxFeePerGas: feeData.maxFeePerGas || 1000000000n,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas || 1000000000n,
    paymasterAndData: '0x',
    signature: '0x'
  };

  // Sign UserOperation
  const userOpHash = getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453);
  userOp.signature = await ownerAccount.signMessage({
    message: { raw: userOpHash }
  });

  return userOp;
}
```

## Approve + Swap Pattern

Most common batch pattern: approve tokens then swap in one transaction:

```typescript
const USDC_ADDRESS = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const AERODROME_ROUTER = '0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43';
const WETH_ADDRESS = '0x4200000000000000000000000000000000000006';

async function approveAndSwap(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  amountIn: bigint
) {
  // Step 1: Encode approve call
  const approveCall: Call = {
    target: USDC_ADDRESS,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function approve(address spender, uint256 amount) external returns (bool)']),
      functionName: 'approve',
      args: [AERODROME_ROUTER, amountIn]
    })
  };

  // Step 2: Encode swap call
  const swapCall: Call = {
    target: AERODROME_ROUTER,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external returns (uint256[] memory amounts)']),
      functionName: 'swapExactTokensForTokens',
      args: [
        amountIn,
        0n, // Set proper slippage in production
        [USDC_ADDRESS, WETH_ADDRESS],
        smartWalletAddress,
        BigInt(Math.floor(Date.now() / 1000) + 1200)
      ]
    })
  };

  // Step 3: Create batch UserOperation
  console.log('Creating approve + swap batch...');
  const userOp = await createBatchUserOp(
    smartWalletAddress,
    [approveCall, swapCall],
    ownerAccount
  );

  // Step 4: Submit and wait
  const userOpHash = await submitUserOperation(userOp);
  console.log('Batch submitted:', userOpHash);

  const receipt = await waitForUserOp(userOpHash);
  console.log('Approve + Swap completed:', receipt.transactionHash);

  return receipt;
}
```

## Multi-Send Pattern

Send tokens to multiple recipients in one transaction:

```typescript
interface Recipient {
  address: Address;
  amount: bigint;
}

async function multiSendUSDC(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  recipients: Recipient[]
) {
  // Create a transfer call for each recipient
  const calls: Call[] = recipients.map(recipient => ({
    target: USDC_ADDRESS,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function transfer(address to, uint256 amount) external returns (bool)']),
      functionName: 'transfer',
      args: [recipient.address, recipient.amount]
    })
  }));

  // Calculate total amount
  const total = recipients.reduce((sum, r) => sum + r.amount, 0n);
  console.log(`Sending ${total} USDC to ${recipients.length} recipients`);

  // Create and submit batch
  const userOp = await createBatchUserOp(smartWalletAddress, calls, ownerAccount);
  const userOpHash = await submitUserOperation(userOp);
  const receipt = await waitForUserOp(userOpHash);

  console.log('Multi-send completed:', receipt.transactionHash);
  return receipt;
}

// Example usage
const recipients: Recipient[] = [
  { address: '0x742d35Cc6634C0532925a3b844Bc454e4438f44e', amount: 10_000000n }, // 10 USDC
  { address: '0x1234567890123456789012345678901234567890', amount: 25_000000n }, // 25 USDC
  { address: '0xabcdefabcdefabcdefabcdefabcdefabcdefabcd', amount: 15_000000n }  // 15 USDC
];

await multiSendUSDC(smartWalletAddress, ownerAccount, recipients);
```

## Complex DeFi Strategy

Combine multiple DeFi operations atomically:

```typescript
const MOONWELL_USDC_MARKET = '0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22';

async function leveragedYieldFarming(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  initialUSDC: bigint
) {
  const calls: Call[] = [];

  // Step 1: Approve USDC to Moonwell
  calls.push({
    target: USDC_ADDRESS,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function approve(address spender, uint256 amount) external returns (bool)']),
      functionName: 'approve',
      args: [MOONWELL_USDC_MARKET, initialUSDC]
    })
  });

  // Step 2: Supply USDC to Moonwell
  calls.push({
    target: MOONWELL_USDC_MARKET,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function mint(uint256 mintAmount) external returns (uint256)']),
      functionName: 'mint',
      args: [initialUSDC]
    })
  });

  // Step 3: Enter markets (enable as collateral)
  calls.push({
    target: '0xfBb21d0380beE3312B33c4353c8936a0F13EF26C', // Moonwell Comptroller
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function enterMarkets(address[] calldata mTokens) external returns (uint256[] memory)']),
      functionName: 'enterMarkets',
      args: [[MOONWELL_USDC_MARKET]]
    })
  });

  // Step 4: Borrow USDC (50% LTV)
  const borrowAmount = initialUSDC / 2n;
  calls.push({
    target: MOONWELL_USDC_MARKET,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function borrow(uint256 borrowAmount) external returns (uint256)']),
      functionName: 'borrow',
      args: [borrowAmount]
    })
  });

  // Step 5: Approve borrowed USDC for swap
  calls.push({
    target: USDC_ADDRESS,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function approve(address spender, uint256 amount) external returns (bool)']),
      functionName: 'approve',
      args: [AERODROME_ROUTER, borrowAmount]
    })
  });

  // Step 6: Swap borrowed USDC for WETH
  calls.push({
    target: AERODROME_ROUTER,
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external returns (uint256[] memory amounts)']),
      functionName: 'swapExactTokensForTokens',
      args: [
        borrowAmount,
        0n,
        [USDC_ADDRESS, WETH_ADDRESS],
        smartWalletAddress,
        BigInt(Math.floor(Date.now() / 1000) + 1200)
      ]
    })
  });

  console.log('Executing leveraged yield farming strategy...');
  console.log('Initial deposit:', initialUSDC / 1_000000n, 'USDC');
  console.log('Borrow amount:', borrowAmount / 1_000000n, 'USDC');
  console.log('Total operations:', calls.length);

  const userOp = await createBatchUserOp(smartWalletAddress, calls, ownerAccount);
  const userOpHash = await submitUserOperation(userOp);
  const receipt = await waitForUserOp(userOpHash);

  console.log('Strategy executed:', receipt.transactionHash);
  return receipt;
}
```

## Batching with Paymaster

Combine batch operations with gasless transactions:

```typescript
async function sponsoredBatchOperation(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  calls: Call[],
  coinbaseApiKey: string
) {
  // Create batch calldata
  const targets = calls.map(c => c.target);
  const values = calls.map(c => c.value);
  const dataArray = calls.map(c => c.data);

  const callData = encodeFunctionData({
    abi: SMART_WALLET_ABI,
    functionName: 'executeBatch',
    args: [targets, values, dataArray]
  });

  // Get nonce and init code
  const nonce = await publicClient.readContract({
    address: ENTRYPOINT_ADDRESS,
    abi: parseAbi(['function getNonce(address sender, uint192 key) external view returns (uint256 nonce)']),
    functionName: 'getNonce',
    args: [smartWalletAddress, 0n]
  });

  const code = await publicClient.getBytecode({ address: smartWalletAddress });
  const initCode = (code && code !== '0x') ? '0x' : getInitCode(ownerAccount.address);

  const feeData = await publicClient.estimateFeesPerGas();

  // Create UserOperation
  let userOp: UserOperation = {
    sender: smartWalletAddress,
    nonce,
    initCode,
    callData,
    callGasLimit: BigInt(calls.length) * 150000n + 200000n,
    verificationGasLimit: 500000n,
    preVerificationGas: 50000n + BigInt(calls.length) * 10000n,
    maxFeePerGas: feeData.maxFeePerGas || 1000000000n,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas || 1000000000n,
    paymasterAndData: '0x',
    signature: '0x'
  };

  // Get dummy signature
  const dummyHash = getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453);
  userOp.signature = await ownerAccount.signMessage({ message: { raw: dummyHash } });

  // Request paymaster sponsorship
  const paymasterData = await getPaymasterData(userOp, coinbaseApiKey);
  userOp.paymasterAndData = paymasterData.paymasterAndData;
  userOp.callGasLimit = paymasterData.callGasLimit;
  userOp.verificationGasLimit = paymasterData.verificationGasLimit;
  userOp.preVerificationGas = paymasterData.preVerificationGas;

  // Final signature
  const finalHash = getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453);
  userOp.signature = await ownerAccount.signMessage({ message: { raw: finalHash } });

  console.log('Submitting sponsored batch operation...');
  const userOpHash = await submitUserOperation(userOp);
  const receipt = await waitForUserOp(userOpHash);

  console.log('Sponsored batch completed:', receipt.transactionHash);
  return receipt;
}
```

## Conditional Execution

Implement try-catch behavior in batches:

```typescript
// Some smart wallets support try-catch execution modes
interface TryCall extends Call {
  allowFailure: boolean; // If true, continue batch even if this call fails
}

async function executeBatchWithFailureHandling(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  calls: TryCall[]
) {
  // Encode with failure flags
  const targets = calls.map(c => c.target);
  const values = calls.map(c => c.value);
  const dataArray = calls.map(c => c.data);
  const allowFailureFlags = calls.map(c => c.allowFailure);

  const callData = encodeFunctionData({
    abi: parseAbi(['function executeBatchWithFlags(address[] calldata dest, uint256[] calldata value, bytes[] calldata func, bool[] calldata allowFailure) external']),
    functionName: 'executeBatchWithFlags',
    args: [targets, values, dataArray, allowFailureFlags]
  });

  // Create and submit UserOperation
  const userOp = await createUserOperation({
    sender: smartWalletAddress,
    callData,
    owner: ownerAccount
  });

  const userOpHash = await submitUserOperation(userOp);
  return await waitForUserOp(userOpHash);
}

// Example: Try to claim rewards from multiple protocols
const claimCalls: TryCall[] = [
  {
    target: '0xProtocol1RewardsContract',
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function claimRewards() external']),
      functionName: 'claimRewards'
    }),
    allowFailure: true // Continue even if no rewards
  },
  {
    target: '0xProtocol2RewardsContract',
    value: 0n,
    data: encodeFunctionData({
      abi: parseAbi(['function harvest() external']),
      functionName: 'harvest'
    }),
    allowFailure: true
  }
];
```

## Common Patterns

**Pattern 1: Approve + Action**
Always batch approvals with the action that requires them. Saves gas and improves UX.

**Pattern 2: Portfolio Rebalancing**
Sell multiple tokens, swap to target allocation, buy new tokens -- all atomically.

**Pattern 3: Emergency Exit**
Claim rewards, withdraw liquidity, swap to stablecoins, bridge to safety -- single transaction.

**Pattern 4: Automated Strategy**
Periodically execute complex multi-step strategies (compound yield, rebalance LP, etc.).

## Gotchas

**Gas Estimation**: Batch operations can consume significant gas. Always overestimate and test thoroughly.

**Atomic Failures**: If ANY call in a batch fails, the ENTIRE batch reverts. Design carefully.

**Approval Amounts**: When batching approve + swap, ensure approval amount covers the swap amount.

**Reentrancy**: Be cautious with batches that call external contracts multiple times.

**Call Ordering**: Order matters. Approvals must come before spends, withdraws before swaps, etc.

**Gas Price Spikes**: Large batches during high gas periods can be very expensive.

**MEV Exposure**: Complex batches may be vulnerable to sandwich attacks. Consider using MEV protection.

**Testing**: Always test batch operations on testnet with realistic conditions before production use.
}