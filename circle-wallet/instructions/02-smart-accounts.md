# Smart Contract Accounts with ERC-4337

Circle's Smart Contract Accounts (SCAs) implement ERC-4337 Account Abstraction on Base, enabling programmable validation logic, gas sponsorship, batch operations, and session-based permissions.

## Account Abstraction Overview

Traditional Ethereum accounts (EOAs) require users to hold ETH for gas and manage private keys directly. ERC-4337 introduces Smart Contract Accounts that:

- Execute transactions through smart contract logic instead of ECDSA signatures
- Enable custom validation rules (multi-sig, social recovery, spending limits)
- Support gas payment in ERC-20 tokens via paymasters
- Allow batching multiple operations into a single transaction
- Provide session keys for delegated, limited-permission access

Unlike EOAs, SCAs separate transaction validation from execution, enabling programmable security policies.

## EntryPoint Contract Architecture

All ERC-4337 operations flow through a singleton EntryPoint contract on Base:

**Base Mainnet EntryPoint**: `0x0576a174D229E3cFA37253523E645A78A0C91B57`

The EntryPoint coordinates the validation and execution flow:

1. Bundler submits UserOperation to EntryPoint
2. EntryPoint calls `validateUserOp` on the Smart Contract Account
3. If validation succeeds, EntryPoint calls `execute` with callData
4. EntryPoint handles gas accounting and paymaster interactions

This architecture ensures atomic validation -> execution with standardized gas handling.

## UserOperation Structure

A UserOperation represents a user's intent to execute an action. Each UserOperation contains:

```typescript
interface UserOperation {
  sender: string;                    // Smart Contract Account address
  nonce: bigint;                     // Anti-replay protection
  initCode: string;                  // Account creation code (empty if exists)
  callData: string;                  // Encoded function call
  callGasLimit: bigint;              // Gas for main execution
  verificationGasLimit: bigint;      // Gas for validation logic
  preVerificationGas: bigint;        // Fixed overhead gas
  maxFeePerGas: bigint;              // Max gas price willing to pay
  maxPriorityFeePerGas: bigint;      // Max priority fee (tip)
  paymasterAndData: string;          // Paymaster address + data
  signature: string;                 // Validation signature
}
```

**Key Fields**:

- `sender`: The SCA address executing the operation
- `nonce`: Prevents replay attacks, increments with each operation
- `initCode`: For counterfactual deployment -- creates account on first use
- `callData`: ABI-encoded function call to execute
- `paymasterAndData`: Enables gas sponsorship (Circle handles this)
- `signature`: Proves authorization according to account's validation logic

## Creating Smart Contract Accounts

Create an SCA wallet via Circle's Developer Controlled Wallets API:

```typescript
import axios from 'axios';

const CIRCLE_API_KEY = process.env.CIRCLE_API_KEY;
const BASE_URL = 'https://api.circle.com/v1/w3s';

const headers = {
  'Authorization': `Bearer ${CIRCLE_API_KEY}`,
  'Content-Type': 'application/json'
};

async function createSmartAccount(entitySecretCiphertext: string) {
  const response = await axios.post(
    `${BASE_URL}/developer/wallets`,
    {
      accountType: 'SCA',
      blockchains: ['BASE-MAINNET'],
      count: 1,
      entitySecretCiphertext
    },
    { headers }
  );

  const wallet = response.data.data.wallets[0];
  return {
    walletId: wallet.id,
    address: wallet.address,
    blockchain: wallet.blockchain
  };
}
```

**Response Structure**:

```json
{
  "data": {
    "wallets": [
      {
        "id": "01933b42-1234-5678-9abc-def012345678",
        "address": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
        "blockchain": "BASE-MAINNET",
        "accountType": "SCA",
        "state": "LIVE",
        "createDate": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

The SCA is deployed lazily -- the address exists but the contract deploys on first transaction.

## Sub Accounts

Sub Accounts are application-specific wallets derived from a main SCA, enabling context isolation:

**Use Cases**:

- Gaming: separate wallet for each game to limit exposure
- DeFi: isolated wallets per protocol (lending, trading, staking)
- Marketplace: buyer vs. seller wallets with different permissions

**Benefits**:

- **Spending Limits**: Cap maximum value per sub-account
- **Session Duration**: Time-limited access (expires after 24 hours)
- **Context Separation**: Compromised sub-account does not affect main account

```typescript
async function createSubAccount(
  mainWalletId: string,
  context: string,
  spendingLimit: string,
  sessionDuration: number
) {
  const response = await axios.post(
    `${BASE_URL}/developer/wallets/${mainWalletId}/subaccounts`,
    {
      context,
      limits: {
        daily: spendingLimit,
        transaction: spendingLimit
      },
      expiresAt: new Date(Date.now() + sessionDuration * 1000).toISOString()
    },
    { headers }
  );

  return response.data.data.subAccount;
}

// Example: Create gaming sub-account with 100 USDC limit
const gamingWallet = await createSubAccount(
  'wallet-id-main',
  'chess-game',
  '100000000', // 100 USDC (6 decimals)
  86400 // 24 hours
);
```

## Session Keys

Session Keys grant limited permissions to third-party services without exposing main account control:

**Permission Controls**:

- **Spend Limit**: Maximum token amount per session
- **Whitelisted Contracts**: Only interact with approved addresses
- **Expiration Timestamp**: Automatic revocation after deadline
- **Operation Types**: Restrict to specific function selectors

```typescript
import { encodeFunctionData, parseUnits } from 'viem';

interface SessionKeyParams {
  sessionPublicKey: `0x${string}`;
  spendLimit: string;
  allowedContracts: `0x${string}`[];
  expiresAt: number;
}

async function createSessionKey(params: SessionKeyParams) {
  const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
  
  // Encode session key registration
  const sessionData = encodeFunctionData({
    abi: [{
      name: 'registerSessionKey',
      type: 'function',
      inputs: [
        { name: 'key', type: 'address' },
        { name: 'limit', type: 'uint256' },
        { name: 'contracts', type: 'address[]' },
        { name: 'expiry', type: 'uint48' }
      ]
    }],
    functionName: 'registerSessionKey',
    args: [
      params.sessionPublicKey,
      parseUnits(params.spendLimit, 6),
      params.allowedContracts,
      params.expiresAt
    ]
  });

  // Submit via Circle API
  const response = await axios.post(
    `${BASE_URL}/developer/transactions/contractExecution`,
    {
      walletId: 'your-wallet-id',
      abiFunctionSignature: 'registerSessionKey(address,uint256,address[],uint48)',
      abiParameters: [
        params.sessionPublicKey,
        params.spendLimit,
        params.allowedContracts,
        params.expiresAt.toString()
      ],
      contractAddress: '0x742d35Cc6634C0532925a3b844Bc454e4438f44e',
      fee: { type: 'level', config: { feeLevel: 'MEDIUM' } }
    },
    { headers }
  );

  return response.data.data.id;
}

// Example: Create session key for NFT marketplace
const sessionTx = await createSessionKey({
  sessionPublicKey: '0x8626f6940E2eb28930eFb4CeF49B2d1F2C9C1199',
  spendLimit: '500000000', // 500 USDC
  allowedContracts: [
    '0x1234567890123456789012345678901234567890' // NFT marketplace
  ],
  expiresAt: Math.floor(Date.now() / 1000) + 7 * 86400 // 7 days
});
```

## Batch Operations

Execute multiple operations atomically in a single UserOperation:

```typescript
import { encodeFunctionData } from 'viem';

const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const NFT_CONTRACT = '0xAbCdEf1234567890AbCdEf1234567890AbCdEf12';

async function executeBatchOperation(
  walletId: string,
  recipient: `0x${string}`,
  usdcAmount: string,
  nftTokenId: string
) {
  // Encode USDC transfer
  const usdcTransferData = encodeFunctionData({
    abi: [{
      name: 'transfer',
      type: 'function',
      inputs: [
        { name: 'to', type: 'address' },
        { name: 'amount', type: 'uint256' }
      ]
    }],
    functionName: 'transfer',
    args: [recipient, parseUnits(usdcAmount, 6)]
  });

  // Encode NFT mint
  const nftMintData = encodeFunctionData({
    abi: [{
      name: 'mint',
      type: 'function',
      inputs: [
        { name: 'to', type: 'address' },
        { name: 'tokenId', type: 'uint256' }
      ]
    }],
    functionName: 'mint',
    args: [recipient, BigInt(nftTokenId)]
  });

  // Submit batch transaction
  const response = await axios.post(
    `${BASE_URL}/developer/transactions/batch`,
    {
      walletId,
      operations: [
        {
          to: USDC_BASE,
          value: '0',
          data: usdcTransferData
        },
        {
          to: NFT_CONTRACT,
          value: '0',
          data: nftMintData
        }
      ],
      fee: { type: 'level', config: { feeLevel: 'MEDIUM' } }
    },
    { headers }
  );

  return response.data.data.id;
}

// Example: Send 50 USDC + mint NFT in single transaction
const batchTxId = await executeBatchOperation(
  'wallet-id',
  '0x9876543210987654321098765432109876543210',
  '50',
  '42'
);

console.log(`Batch transaction: ${batchTxId}`);
// Both operations succeed or both fail atomically
```

## Validation Flow

ERC-4337 validation and execution sequence:

```
1. Bundler receives UserOperation from client
2. Bundler calls EntryPoint.handleOps([userOp])
3. EntryPoint validates UserOperation:
   - Check nonce is valid
   - Call account.validateUserOp(userOp, userOpHash, missingAccountFunds)
   - Account checks signature matches validation logic
   - If paymaster exists, call paymaster.validatePaymasterUserOp()
4. If validation succeeds, EntryPoint executes:
   - Call account.execute(callData)
   - Account performs requested operation
   - Refund excess gas to paymaster or account
5. Emit UserOperationEvent with execution result
```

**Validation Function in SCA**:

```solidity
function validateUserOp(
    UserOperation calldata userOp,
    bytes32 userOpHash,
    uint256 missingAccountFunds
) external returns (uint256 validationData) {
    // Only EntryPoint can call
    require(msg.sender == ENTRYPOINT, "Only EntryPoint");
    
    // Verify signature
    bytes32 hash = userOpHash.toEthSignedMessageHash();
    address signer = hash.recover(userOp.signature);
    require(isOwner(signer), "Invalid signature");
    
    // Pay EntryPoint if needed
    if (missingAccountFunds > 0) {
        (bool success,) = payable(msg.sender).call{
            value: missingAccountFunds
        }("");
        require(success, "Payment failed");
    }
    
    // Return validation success
    return 0; // SIG_VALIDATION_SUCCEEDED
}
```

## Gas Sponsorship

Circle's paymasters sponsor gas fees for Smart Contract Accounts:

```typescript
async function sponsoredTransfer(
  walletId: string,
  recipientAddress: string,
  amount: string
) {
  const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';

  const response = await axios.post(
    `${BASE_URL}/developer/transactions/transfer`,
    {
      walletId,
      tokenId: USDC_BASE,
      destination: recipientAddress,
      amounts: [amount],
      fee: {
        type: 'level',
        config: {
          feeLevel: 'MEDIUM',
          sponsor: true // Circle pays gas
        }
      }
    },
    { headers }
  );

  return response.data.data;
}

// User sends USDC without holding ETH
const tx = await sponsoredTransfer(
  'wallet-id',
  '0x5555555555555555555555555555555555555555',
  '25000000' // 25 USDC
);
```

The paymaster validates and pays gas, charging Circle's account instead of the user's wallet.

## Integration with Smart Wallet Skill

Smart Contract Accounts work seamlessly with Circle's Smart Wallet authentication:

```typescript
import { SmartWalletAuth } from '@circle/smart-wallet';

// Initialize Smart Wallet with SCA
const smartWallet = new SmartWalletAuth({
  apiKey: CIRCLE_API_KEY,
  appId: 'your-app-id',
  accountType: 'SCA'
});

// User logs in -> creates SCA if needed
const session = await smartWallet.login({
  email: 'user@example.com',
  chain: 'base-mainnet'
});

// Execute batch operation via authenticated session
const batchResult = await session.executeBatch([
  {
    to: USDC_BASE,
    data: transferData,
    value: '0'
  },
  {
    to: NFT_CONTRACT,
    data: mintData,
    value: '0'
  }
]);
```

See [Smart Wallet Integration](./smart-wallet.md) for complete authentication flows.

## Integration with X402 Payments

Smart Contract Accounts enable X402 micropayments with session keys:

```typescript
// Create session key for X402 content access
const x402SessionKey = await createSessionKey({
  sessionPublicKey: contentProviderKey,
  spendLimit: '10000000', // 10 USDC for content
  allowedContracts: [
    X402_PAYMENT_PROCESSOR
  ],
  expiresAt: Math.floor(Date.now() / 1000) + 30 * 86400 // 30 days
});

// Provider can now charge for content without user approval each time
```

Refer to [X402 Payment Protocol](./x402.md) for full implementation details.

## Security Considerations

**Nonce Management**: Ensure nonces increment correctly to prevent replay attacks.

**Session Key Limits**: Always set strict spending limits and expiration times.

**Contract Whitelisting**: Only allow session keys to interact with audited contracts.

**Sub-Account Isolation**: Use separate sub-accounts for high-risk operations.

**Signature Verification**: Validate all signatures match authorized signers before execution.

## Next Steps

- Review [Smart Wallet Setup](./smart-wallet.md) for authentication integration
- Implement [X402 Payments](./x402.md) with session key delegation
- Explore [Base Network Configuration](./01-base-setup.md) for network parameters
- Test batch operations on Base Sepolia testnet before mainnet deployment

Smart Contract Accounts provide powerful programmability while maintaining security through Circle's managed infrastructure and ERC-4337's standardized validation flow.