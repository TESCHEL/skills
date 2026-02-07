# Session Keys for Delegated Smart Wallet Access

## Overview

Session keys enable temporary, limited permissions for smart wallets. They allow dApps, AI agents, or other services to execute specific actions on behalf of a user without requiring constant approval or access to master keys.

Session keys are crucial for AI agents because they provide:
- Time-bounded permissions (auto-revocation)
- Spending limits per session
- Contract-specific authorization
- Function-level granularity
- Automatic expiration

## Session Key Architecture

Session keys work through a permission system installed as a module on the smart wallet:

```typescript
interface SessionKeyPermission {
  signer: Address;           // The session key address
  validUntil: number;        // Unix timestamp when permission expires
  validAfter: number;        // Unix timestamp when permission becomes valid
  sessionValidationModule: Address; // Module that validates this session
  sessionKeyData: Hex;       // Encoded permission parameters
}

interface SessionKeyData {
  targetContract: Address;   // Allowed contract to interact with
  functionSelector: Hex;     // Allowed function (or 0x00000000 for any)
  valueLimit: bigint;        // Max ETH value per call
  spendingLimit: bigint;     // Max tokens to spend total
  spentAmount: bigint;       // Already spent amount
}
```

## Installing Session Key Module

First, install the session key validation module on your smart wallet:

```typescript
import { encodeFunctionData, parseAbi } from 'viem';

const SESSION_KEY_MODULE = '0x...' // Session key module address

const SMART_WALLET_ABI = parseAbi([
  'function execute(address dest, uint256 value, bytes calldata func) external',
  'function executeBatch(address[] calldata dest, uint256[] calldata value, bytes[] calldata func) external',
  'function installModule(uint256 moduleTypeId, address module, bytes calldata initData) external'
]);

// Module type ID: 1 = Validator, 2 = Executor, 3 = Fallback, 4 = Hook
const VALIDATOR_MODULE_TYPE = 1n;

async function installSessionKeyModule(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>
) {
  // Encode the module installation call
  const installCallData = encodeFunctionData({
    abi: SMART_WALLET_ABI,
    functionName: 'installModule',
    args: [
      VALIDATOR_MODULE_TYPE,
      SESSION_KEY_MODULE,
      '0x' // Empty init data
    ]
  });

  // Create UserOperation to install module
  const userOp = await createUserOperation({
    sender: smartWalletAddress,
    callData: installCallData,
    owner: ownerAccount
  });

  const userOpHash = await submitUserOperation(userOp);
  await waitForUserOp(userOpHash);
  
  console.log('Session key module installed');
}
```

## Granting Session Keys

Create and authorize a new session key with specific permissions:

```typescript
import { privateKeyToAccount } from 'viem/accounts';
import { generatePrivateKey } from 'viem/accounts';

const SESSION_KEY_MANAGER_ABI = parseAbi([
  'function setSessionKey(address sessionKey, uint48 validUntil, uint48 validAfter, bytes calldata sessionKeyData) external'
]);

async function grantSessionKey(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  config: {
    targetContract: Address;
    functionSelector: Hex;
    durationSeconds: number;
    spendingLimit: bigint;
    valueLimit?: bigint;
  }
) {
  // Generate new session key
  const sessionPrivateKey = generatePrivateKey();
  const sessionKeyAccount = privateKeyToAccount(sessionPrivateKey);
  
  console.log('Generated Session Key:', sessionKeyAccount.address);
  console.log('Private Key (SAVE SECURELY):', sessionPrivateKey);

  // Calculate validity period
  const validAfter = Math.floor(Date.now() / 1000);
  const validUntil = validAfter + config.durationSeconds;

  // Encode session key data
  const sessionKeyData = encodeAbiParameters(
    parseAbiParameters('address, bytes4, uint256, uint256'),
    [
      config.targetContract,
      config.functionSelector,
      config.valueLimit || 0n,
      config.spendingLimit
    ]
  );

  // Create call to set session key
  const setSessionKeyCall = encodeFunctionData({
    abi: SESSION_KEY_MANAGER_ABI,
    functionName: 'setSessionKey',
    args: [
      sessionKeyAccount.address,
      validUntil,
      validAfter,
      sessionKeyData
    ]
  });

  // Execute via smart wallet
  const executeCallData = encodeFunctionData({
    abi: SMART_WALLET_ABI,
    functionName: 'execute',
    args: [SESSION_KEY_MODULE, 0n, setSessionKeyCall]
  });

  const userOp = await createUserOperation({
    sender: smartWalletAddress,
    callData: executeCallData,
    owner: ownerAccount
  });

  const userOpHash = await submitUserOperation(userOp);
  await waitForUserOp(userOpHash);

  return {
    sessionKey: sessionKeyAccount.address,
    privateKey: sessionPrivateKey,
    validUntil,
    validAfter,
    permissions: config
  };
}
```

## Using Session Keys

Execute transactions using a session key instead of the master key:

```typescript
async function executeWithSessionKey(
  smartWalletAddress: Address,
  sessionKeyAccount: ReturnType<typeof privateKeyToAccount>,
  targetContract: Address,
  callData: Hex
) {
  // Create UserOperation with session key signature
  const userOp = await createUserOperation({
    sender: smartWalletAddress,
    callData: encodeFunctionData({
      abi: SMART_WALLET_ABI,
      functionName: 'execute',
      args: [targetContract, 0n, callData]
    }),
    owner: sessionKeyAccount // Use session key instead of master key
  });

  // Sign with session key
  const userOpHash = getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453);
  const sessionSignature = await sessionKeyAccount.signMessage({
    message: { raw: userOpHash }
  });

  // Wrap signature with session key identifier
  userOp.signature = encodeAbiParameters(
    parseAbiParameters('address, bytes'),
    [sessionKeyAccount.address, sessionSignature]
  );

  const userOpHash2 = await submitUserOperation(userOp);
  return await waitForUserOp(userOpHash2);
}
```

## Example: Trading Session Key

Grant a session key permission to swap tokens on Aerodrome:

```typescript
const AERODROME_ROUTER = '0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43';
const USDC_ADDRESS = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';

// Grant session key for swapping up to 100 USDC for 24 hours
const tradingSession = await grantSessionKey(
  smartWalletAddress,
  ownerAccount,
  {
    targetContract: AERODROME_ROUTER,
    functionSelector: '0x38ed1739', // swapExactTokensForTokens selector
    durationSeconds: 86400, // 24 hours
    spendingLimit: 100_000000n, // 100 USDC (6 decimals)
    valueLimit: 0n // No ETH value allowed
  }
);

console.log('Trading session created:');
console.log('  Session Key:', tradingSession.sessionKey);
console.log('  Valid Until:', new Date(tradingSession.validUntil * 1000));
console.log('  Max Spending: 100 USDC');

// Later, use the session key to execute a swap
const swapCallData = encodeFunctionData({
  abi: parseAbi(['function swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin, address[] calldata path, address to, uint256 deadline) external returns (uint256[] memory amounts)']),
  functionName: 'swapExactTokensForTokens',
  args: [
    50_000000n, // 50 USDC
    0n, // Accept any amount (set properly in production)
    [USDC_ADDRESS, '0x4200000000000000000000000000000000000006'], // USDC -> WETH
    smartWalletAddress,
    BigInt(Math.floor(Date.now() / 1000) + 1200) // 20 min deadline
  ]
});

const sessionKeyAccount = privateKeyToAccount(tradingSession.privateKey);
const receipt = await executeWithSessionKey(
  smartWalletAddress,
  sessionKeyAccount,
  AERODROME_ROUTER,
  swapCallData
);

console.log('Swap executed:', receipt.transactionHash);
```

## Spending Limit Tracking

Track spending to ensure limits are enforced:

```typescript
const SESSION_KEY_STORAGE_ABI = parseAbi([
  'function getSessionKeyData(address sessionKey) external view returns (uint48 validUntil, uint48 validAfter, uint256 spentAmount, bytes memory data)'
]);

async function checkSessionKeySpending(
  smartWalletAddress: Address,
  sessionKeyAddress: Address
): Promise<{ spent: bigint; limit: bigint; remaining: bigint }> {
  const result = await publicClient.readContract({
    address: SESSION_KEY_MODULE,
    abi: SESSION_KEY_STORAGE_ABI,
    functionName: 'getSessionKeyData',
    args: [sessionKeyAddress]
  });

  const [validUntil, validAfter, spentAmount, data] = result;
  
  // Decode session key data to get limit
  const [targetContract, functionSelector, valueLimit, spendingLimit] = decodeAbiParameters(
    parseAbiParameters('address, bytes4, uint256, uint256'),
    data
  );

  return {
    spent: spentAmount,
    limit: spendingLimit,
    remaining: spendingLimit - spentAmount
  };
}
```

## Revoking Session Keys

Revoke a session key before expiration:

```typescript
const SESSION_KEY_REVOCATION_ABI = parseAbi([
  'function revokeSessionKey(address sessionKey) external'
]);

async function revokeSessionKey(
  smartWalletAddress: Address,
  ownerAccount: ReturnType<typeof privateKeyToAccount>,
  sessionKeyAddress: Address
) {
  const revokeCall = encodeFunctionData({
    abi: SESSION_KEY_REVOCATION_ABI,
    functionName: 'revokeSessionKey',
    args: [sessionKeyAddress]
  });

  const executeCallData = encodeFunctionData({
    abi: SMART_WALLET_ABI,
    functionName: 'execute',
    args: [SESSION_KEY_MODULE, 0n, revokeCall]
  });

  const userOp = await createUserOperation({
    sender: smartWalletAddress,
    callData: executeCallData,
    owner: ownerAccount
  });

  const userOpHash = await submitUserOperation(userOp);
  await waitForUserOp(userOpHash);
  
  console.log('Session key revoked:', sessionKeyAddress);
}
```

## Common Patterns

**Pattern 1: AI Agent Session**
Create long-lived session keys for AI agents with low spending limits and specific contract permissions.

**Pattern 2: dApp Integration**
Grant session keys to dApps for specific actions (trading, lending, gaming) without exposing master keys.

**Pattern 3: Tiered Permissions**
Create multiple session keys with different limits: micro-transactions, standard, high-value.

**Pattern 4: Auto-Renewal**
Implement a system where session keys near expiration can be automatically renewed by the owner.

## Gotchas

**Expiration Checks**: Always verify validUntil before using a session key. Expired keys will cause transaction failures.

**Spending Tracking**: The module tracks spending on-chain. Off-chain tracking may drift if transactions revert.

**Function Selectors**: Using 0x00000000 allows ANY function on the target contract. Be very careful with wildcards.

**Module Compatibility**: Not all smart wallet implementations support session keys. Verify compatibility before deployment.

**Key Storage**: Session key private keys must be stored securely. If compromised, revoke immediately.

**Gas Costs**: Session key validation adds gas overhead. Budget approximately 50k-100k extra gas per operation.