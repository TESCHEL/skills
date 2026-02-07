# Paymasters for Gasless Transactions on Base

## Overview

Paymasters are ERC-4337 contracts that sponsor gas fees for UserOperations. This enables gasless transactions where users can interact with smart contracts without holding ETH for gas. The Coinbase Paymaster on Base allows eligible transactions to be executed with sponsored gas fees.

Paymasters are essential for:
- Onboarding new users without requiring ETH
- AI agents executing transactions without managing gas balances
- dApps improving UX by removing gas friction
- Subscription-based services where the protocol pays gas

## Paymaster Architecture

Paymasters implement the IPaymaster interface and are called by the EntryPoint during UserOperation validation:

```typescript
interface IPaymaster {
  // Validate that the paymaster will pay for this UserOperation
  // Returns context data and validation data (validUntil, validAfter)
  function validatePaymasterUserOp(
    UserOperation calldata userOp,
    bytes32 userOpHash,
    uint256 maxCost
  ) external returns (bytes memory context, uint256 validationData);

  // Called after the main execution (if validation succeeded)
  function postOp(
    PostOpMode mode,
    bytes calldata context,
    uint256 actualGasCost
  ) external;
}
```

Paymaster flow:
1. User creates UserOperation with paymasterAndData field populated
2. Bundler simulates operation, calling validatePaymasterUserOp
3. If validation succeeds, bundler includes operation in bundle
4. EntryPoint executes operation, paymaster pays gas
5. EntryPoint calls postOp on paymaster for accounting

## Coinbase Paymaster on Base

Coinbase provides a paymaster service for eligible transactions on Base:

```typescript
const COINBASE_PAYMASTER_URL = 'https://api.developer.coinbase.com/rpc/v1/base/paymaster';

interface PaymasterRequest {
  jsonrpc: '2.0';
  id: number;
  method: 'pm_sponsorUserOperation';
  params: [
    {
      userOperation: UserOperation;
      entryPoint: Address;
      context: {
        policyId?: string; // Optional policy for spending limits
      };
    }
  ];
}

interface PaymasterResponse {
  jsonrpc: '2.0';
  id: number;
  result: {
    paymasterAndData: Hex;
    preVerificationGas: Hex;
    verificationGasLimit: Hex;
    callGasLimit: Hex;
  };
}
```

## Requesting Paymaster Sponsorship

Get paymaster signature and gas estimates:

```typescript
import { createPublicClient, http } from 'viem';
import { base } from 'viem/chains';

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

async function getPaymasterData(
  userOp: UserOperation,
  apiKey: string,
  policyId?: string
): Promise<{
  paymasterAndData: Hex;
  preVerificationGas: bigint;
  verificationGasLimit: bigint;
  callGasLimit: bigint;
}> {
  const response = await fetch(COINBASE_PAYMASTER_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'pm_sponsorUserOperation',
      params: [{
        userOperation: {
          sender: userOp.sender,
          nonce: `0x${userOp.nonce.toString(16)}`,
          initCode: userOp.initCode,
          callData: userOp.callData,
          callGasLimit: `0x${userOp.callGasLimit.toString(16)}`,
          verificationGasLimit: `0x${userOp.verificationGasLimit.toString(16)}`,
          preVerificationGas: `0x${userOp.preVerificationGas.toString(16)}`,
          maxFeePerGas: `0x${userOp.maxFeePerGas.toString(16)}`,
          maxPriorityFeePerGas: `0x${userOp.maxPriorityFeePerGas.toString(16)}`,
          paymasterAndData: '0x',
          signature: userOp.signature
        },
        entryPoint: ENTRYPOINT_ADDRESS,
        context: policyId ? { policyId } : {}
      }]
    })
  });

  const result = await response.json();

  if (result.error) {
    throw new Error(`Paymaster error: ${result.error.message}`);
  }

  return {
    paymasterAndData: result.result.paymasterAndData,
    preVerificationGas: BigInt(result.result.preVerificationGas),
    verificationGasLimit: BigInt(result.result.verificationGasLimit),
    callGasLimit: BigInt(result.result.callGasLimit)
  };
}
```

## Creating Sponsored UserOperations

Complete flow for creating a gasless transaction:

```typescript
async function createSponsoredUserOp(
  smartWalletAddress: Address,
  targetContract: Address,
  callData: Hex,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  coinbaseApiKey: string
): Promise<UserOperation> {
  // 1. Get nonce
  const nonce = await publicClient.readContract({
    address: ENTRYPOINT_ADDRESS,
    abi: [{
      name: 'getNonce',
      type: 'function',
      stateMutability: 'view',
      inputs: [
        { name: 'sender', type: 'address' },
        { name: 'key', type: 'uint192' }
      ],
      outputs: [{ name: 'nonce', type: 'uint256' }]
    }],
    functionName: 'getNonce',
    args: [smartWalletAddress, 0n]
  });

  // 2. Check if wallet is deployed
  const code = await publicClient.getBytecode({ address: smartWalletAddress });
  const initCode = (code && code !== '0x') ? '0x' : getInitCode(ownerAccount.address);

  // 3. Encode the call
  const executeCallData = encodeFunctionData({
    abi: parseAbi(['function execute(address dest, uint256 value, bytes calldata func) external']),
    functionName: 'execute',
    args: [targetContract, 0n, callData]
  });

  // 4. Get current gas price
  const feeData = await publicClient.estimateFeesPerGas();

  // 5. Create initial UserOperation with dummy gas values
  let userOp: UserOperation = {
    sender: smartWalletAddress,
    nonce,
    initCode,
    callData: executeCallData,
    callGasLimit: 100000n,
    verificationGasLimit: 500000n,
    preVerificationGas: 50000n,
    maxFeePerGas: feeData.maxFeePerGas || 1000000000n,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas || 1000000000n,
    paymasterAndData: '0x',
    signature: '0x'
  };

  // 6. Get dummy signature for gas estimation
  const dummySignature = await ownerAccount.signMessage({
    message: { raw: getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453) }
  });
  userOp.signature = dummySignature;

  // 7. Request paymaster sponsorship
  const paymasterData = await getPaymasterData(userOp, coinbaseApiKey);

  // 8. Update UserOperation with paymaster data
  userOp.paymasterAndData = paymasterData.paymasterAndData;
  userOp.callGasLimit = paymasterData.callGasLimit;
  userOp.verificationGasLimit = paymasterData.verificationGasLimit;
  userOp.preVerificationGas = paymasterData.preVerificationGas;

  // 9. Sign final UserOperation
  const finalHash = getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453);
  userOp.signature = await ownerAccount.signMessage({
    message: { raw: finalHash }
  });

  return userOp;
}
```

## Example: Gasless Token Swap

Execute a swap on Aerodrome without paying gas:

```typescript
const AERODROME_ROUTER = '0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43';
const USDC_ADDRESS = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const WETH_ADDRESS = '0x4200000000000000000000000000000000000006';

async function gaslessSwap(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  amountIn: bigint,
  coinbaseApiKey: string
) {
  // 1. Prepare swap calldata
  const swapCallData = encodeFunctionData({
    abi: parseAbi(['function swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external returns (uint256[] memory amounts)']),
    functionName: 'swapExactTokensForTokens',
    args: [
      amountIn,
      0n, // Set proper slippage in production
      [USDC_ADDRESS, WETH_ADDRESS],
      smartWalletAddress,
      BigInt(Math.floor(Date.now() / 1000) + 1200)
    ]
  });

  // 2. Create sponsored UserOperation
  console.log('Creating sponsored swap operation...');
  const userOp = await createSponsoredUserOp(
    smartWalletAddress,
    AERODROME_ROUTER,
    swapCallData,
    ownerAccount,
    coinbaseApiKey
  );

  // 3. Submit to bundler
  console.log('Submitting gasless transaction...');
  const userOpHash = await submitUserOperation(userOp);
  console.log('UserOp Hash:', userOpHash);

  // 4. Wait for confirmation
  const receipt = await waitForUserOp(userOpHash);
  console.log('Swap completed without gas payment!');
  console.log('Transaction:', receipt.transactionHash);

  return receipt;
}
```

## Paymaster Policies

Create spending policies for controlled sponsorship:

```typescript
interface PaymasterPolicy {
  policyId: string;
  name: string;
  dailySpendingLimit: bigint; // In wei
  perTransactionLimit: bigint;
  allowedContracts: Address[];
  allowedFunctions: Hex[];
}

async function createPaymasterPolicy(
  apiKey: string,
  policy: Omit<PaymasterPolicy, 'policyId'>
): Promise<string> {
  const response = await fetch('https://api.developer.coinbase.com/rpc/v1/base/policies', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      name: policy.name,
      dailySpendingLimit: policy.dailySpendingLimit.toString(),
      perTransactionLimit: policy.perTransactionLimit.toString(),
      allowedContracts: policy.allowedContracts,
      allowedFunctions: policy.allowedFunctions
    })
  });

  const result = await response.json();
  return result.policyId;
}

// Example: Create policy for DeFi operations
const defiPolicyId = await createPaymasterPolicy(
  coinbaseApiKey,
  {
    name: 'DeFi Operations Policy',
    dailySpendingLimit: parseEther('0.1'), // Max 0.1 ETH gas per day
    perTransactionLimit: parseEther('0.01'), // Max 0.01 ETH per tx
    allowedContracts: [
      AERODROME_ROUTER,
      '0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22' // Moonwell USDC
    ],
    allowedFunctions: [
      '0x38ed1739', // swapExactTokensForTokens
      '0xa0712d68'  // mint
    ]
  }
);
```

## Verifying Paymaster Balance

Check paymaster deposit in EntryPoint:

```typescript
async function getPaymasterBalance(
  paymasterAddress: Address
): Promise<bigint> {
  const balance = await publicClient.readContract({
    address: ENTRYPOINT_ADDRESS,
    abi: parseAbi(['function balanceOf(address account) external view returns (uint256)']),
    functionName: 'balanceOf',
    args: [paymasterAddress]
  });

  return balance;
}

async function addPaymasterDeposit(
  paymasterAddress: Address,
  amount: bigint,
  depositorAccount: ReturnType<typeof privateKeyToAccount>
) {
  const walletClient = createWalletClient({
    account: depositorAccount,
    chain: base,
    transport: http('https://mainnet.base.org')
  });

  const hash = await walletClient.writeContract({
    address: ENTRYPOINT_ADDRESS,
    abi: parseAbi(['function depositTo(address account) external payable']),
    functionName: 'depositTo',
    args: [paymasterAddress],
    value: amount
  });

  await publicClient.waitForTransactionReceipt({ hash });
  console.log('Deposited', amount, 'wei to paymaster');
}
```

## Common Patterns

**Pattern 1: Conditional Sponsorship**
Only sponsor transactions that meet certain criteria (new users, specific actions, value thresholds).

**Pattern 2: Token Payment**
Some paymasters accept ERC-20 tokens instead of ETH. User pays in USDC, paymaster pays ETH gas.

**Pattern 3: Subscription Model**
Users pay monthly subscription, get unlimited sponsored transactions within policy limits.

**Pattern 4: Cross-Subsidization**
Protocol sponsors gas for user acquisition, recoups costs through other revenue streams.

## Gotchas

**Eligibility**: Not all operations are eligible for Coinbase Paymaster sponsorship. Check API documentation for requirements.

**Rate Limits**: Paymaster APIs have rate limits. Implement exponential backoff for retries.

**Gas Price Volatility**: During high gas prices, paymasters may reject operations to prevent excessive costs.

**Policy Violations**: Operations violating paymaster policies will be rejected during validation.

**Signature Order**: Sign UserOperation AFTER getting paymaster data. Changing paymasterAndData invalidates signature.

**Deposit Management**: Self-hosted paymasters must maintain sufficient deposit in EntryPoint. Monitor balance closely.

**Validation Gas**: Paymaster validation consumes gas. Budget 50k-100k extra verificationGasLimit.