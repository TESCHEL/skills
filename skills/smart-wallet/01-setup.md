# Setting Up Coinbase Smart Wallets on Base

## Overview

This lesson covers deploying and initializing Coinbase Smart Wallets on Base using ERC-4337 account abstraction. You will learn the UserOperation structure, how to interact with the EntryPoint contract, and how to deterministically deploy smart wallets.

Coinbase Smart Wallets are ERC-4337 compatible smart contract accounts that provide enhanced security and user experience features beyond traditional externally owned accounts (EOAs).

## ERC-4337 Architecture

The ERC-4337 standard introduces account abstraction without requiring protocol changes to Ethereum/Base. Key components:

**EntryPoint Contract** (0x0576a174D229E3cFA37253523E645A78A0C91B57):
- Singleton contract that handles all UserOperations
- Validates and executes operations atomically
- Manages reputation of paymasters and aggregators
- Emits events for transaction tracking

**UserOperation Structure**:
```typescript
interface UserOperation {
  sender: Address;              // Smart wallet address
  nonce: bigint;               // Anti-replay parameter
  initCode: Hex;               // Factory address + deployment calldata (empty if deployed)
  callData: Hex;               // Actual operation to execute
  callGasLimit: bigint;        // Gas limit for the main execution
  verificationGasLimit: bigint; // Gas limit for verification
  preVerificationGas: bigint;  // Gas for bundler compensation
  maxFeePerGas: bigint;        // Like EIP-1559
  maxPriorityFeePerGas: bigint; // Like EIP-1559
  paymasterAndData: Hex;       // Paymaster address + data (empty if sender pays)
  signature: Hex;              // Signature over userOp hash
}
```

## Deploying a Smart Wallet

Smart wallets are deployed deterministically using CREATE2, allowing you to know the address before deployment.

```typescript
import { createPublicClient, createWalletClient, http, encodeFunctionData, concat, pad } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

// Coinbase Smart Wallet Factory (example address)
const FACTORY_ADDRESS = '0x0BA5ED0c6AA8c49038F819E587E2633c4A9F428a';
const ENTRYPOINT_ADDRESS = '0x0576a174D229E3cFA37253523E645A78A0C91B57';

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
});

// Owner account that will control the smart wallet
const owner = privateKeyToAccount('0x...');

const walletClient = createWalletClient({
  account: owner,
  chain: base,
  transport: http('https://mainnet.base.org')
});

// Calculate deterministic smart wallet address
async function getSmartWalletAddress(
  ownerAddress: Address,
  salt: bigint = 0n
): Promise<Address> {
  // Implementation depends on factory contract
  // Typically: keccak256(0xff ++ factory ++ salt ++ keccak256(initCode))
  const initCode = encodeFunctionData({
    abi: [{
      name: 'createAccount',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'owner', type: 'address' },
        { name: 'salt', type: 'uint256' }
      ],
      outputs: [{ name: 'account', type: 'address' }]
    }],
    functionName: 'createAccount',
    args: [ownerAddress, salt]
  });

  // Call factory to get predicted address
  const address = await publicClient.readContract({
    address: FACTORY_ADDRESS,
    abi: [{
      name: 'getAddress',
      type: 'function',
      stateMutability: 'view',
      inputs: [
        { name: 'owner', type: 'address' },
        { name: 'salt', type: 'uint256' }
      ],
      outputs: [{ name: '', type: 'address' }]
    }],
    functionName: 'getAddress',
    args: [ownerAddress, salt]
  });

  return address;
}

// Create UserOperation for deployment
async function createDeploymentUserOp(
  ownerAddress: Address,
  salt: bigint = 0n
): Promise<UserOperation> {
  const smartWalletAddress = await getSmartWalletAddress(ownerAddress, salt);

  // Check if already deployed
  const code = await publicClient.getBytecode({ address: smartWalletAddress });
  const isDeployed = code !== undefined && code !== '0x';

  // initCode is factory address + createAccount calldata
  const initCode = isDeployed ? '0x' : concat([
    FACTORY_ADDRESS,
    encodeFunctionData({
      abi: [{
        name: 'createAccount',
        type: 'function',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'owner', type: 'address' },
          { name: 'salt', type: 'uint256' }
        ],
        outputs: [{ name: 'account', type: 'address' }]
      }],
      functionName: 'createAccount',
      args: [ownerAddress, salt]
    })
  ]);

  // For deployment, callData can be empty or initialization call
  const callData = '0x';

  // Get nonce from EntryPoint
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

  return {
    sender: smartWalletAddress,
    nonce,
    initCode,
    callData,
    callGasLimit: 100000n,
    verificationGasLimit: 500000n,
    preVerificationGas: 50000n,
    maxFeePerGas: 1000000000n, // 1 gwei
    maxPriorityFeePerGas: 1000000000n,
    paymasterAndData: '0x',
    signature: '0x' // Will be filled after signing
  };
}
```

## Signing UserOperations

UserOperations must be signed by the smart wallet owner:

```typescript
import { keccak256, encodeAbiParameters, parseAbiParameters } from 'viem';

function getUserOpHash(
  userOp: UserOperation,
  entryPoint: Address,
  chainId: number
): Hex {
  // Hash the UserOperation according to ERC-4337
  const packedUserOp = keccak256(
    encodeAbiParameters(
      parseAbiParameters('address, uint256, bytes32, bytes32, uint256, uint256, uint256, uint256, uint256, bytes32'),
      [
        userOp.sender,
        userOp.nonce,
        keccak256(userOp.initCode),
        keccak256(userOp.callData),
        userOp.callGasLimit,
        userOp.verificationGasLimit,
        userOp.preVerificationGas,
        userOp.maxFeePerGas,
        userOp.maxPriorityFeePerGas,
        keccak256(userOp.paymasterAndData)
      ]
    )
  );

  // Wrap in EntryPoint-specific hash
  return keccak256(
    encodeAbiParameters(
      parseAbiParameters('bytes32, address, uint256'),
      [packedUserOp, entryPoint, BigInt(chainId)]
    )
  );
}

async function signUserOp(
  userOp: UserOperation,
  ownerAccount: ReturnType<typeof privateKeyToAccount>
): Promise<UserOperation> {
  const userOpHash = getUserOpHash(userOp, ENTRYPOINT_ADDRESS, 8453);
  const signature = await ownerAccount.signMessage({
    message: { raw: userOpHash }
  });

  return {
    ...userOp,
    signature
  };
}
```

## Submitting to Bundler

UserOperations are submitted to bundlers who package them into transactions:

```typescript
async function submitUserOperation(
  userOp: UserOperation
): Promise<Hex> {
  // Use a bundler RPC endpoint (Pimlico, Stackup, etc.)
  const bundlerUrl = 'https://api.pimlico.io/v1/base/rpc?apikey=YOUR_KEY';

  const response = await fetch(bundlerUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'eth_sendUserOperation',
      params: [
        {
          sender: userOp.sender,
          nonce: `0x${userOp.nonce.toString(16)}`,
          initCode: userOp.initCode,
          callData: userOp.callData,
          callGasLimit: `0x${userOp.callGasLimit.toString(16)}`,
          verificationGasLimit: `0x${userOp.verificationGasLimit.toString(16)}`,
          preVerificationGas: `0x${userOp.preVerificationGas.toString(16)}`,
          maxFeePerGas: `0x${userOp.maxFeePerGas.toString(16)}`,
          maxPriorityFeePerGas: `0x${userOp.maxPriorityFeePerGas.toString(16)}`,
          paymasterAndData: userOp.paymasterAndData,
          signature: userOp.signature
        },
        ENTRYPOINT_ADDRESS
      ]
    })
  });

  const result = await response.json();
  return result.result; // Returns userOpHash
}

// Wait for UserOperation to be mined
async function waitForUserOp(
  userOpHash: Hex
): Promise<{ transactionHash: Hex; success: boolean }> {
  const bundlerUrl = 'https://api.pimlico.io/v1/base/rpc?apikey=YOUR_KEY';

  while (true) {
    const response = await fetch(bundlerUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        id: 1,
        method: 'eth_getUserOperationReceipt',
        params: [userOpHash]
      })
    });

    const result = await response.json();
    if (result.result) {
      return {
        transactionHash: result.result.receipt.transactionHash,
        success: result.result.success
      };
    }

    await new Promise(resolve => setTimeout(resolve, 2000));
  }
}
```

## Complete Deployment Example

```typescript
async function deploySmartWallet(ownerAccount: ReturnType<typeof privateKeyToAccount>) {
  console.log('Calculating smart wallet address...');
  const smartWalletAddress = await getSmartWalletAddress(ownerAccount.address);
  console.log('Smart Wallet Address:', smartWalletAddress);

  console.log('Creating deployment UserOperation...');
  let userOp = await createDeploymentUserOp(ownerAccount.address);

  console.log('Signing UserOperation...');
  userOp = await signUserOp(userOp, ownerAccount);

  console.log('Submitting to bundler...');
  const userOpHash = await submitUserOperation(userOp);
  console.log('UserOp Hash:', userOpHash);

  console.log('Waiting for confirmation...');
  const receipt = await waitForUserOp(userOpHash);
  console.log('Transaction Hash:', receipt.transactionHash);
  console.log('Success:', receipt.success);

  return smartWalletAddress;
}
```

## Common Patterns

**Pattern 1: Lazy Deployment**
Do not deploy the smart wallet until the first transaction. Include initCode in the first UserOperation only.

**Pattern 2: Multi-Owner Wallets**
Some smart wallet implementations support multiple owners. Configure during deployment via callData initialization.

**Pattern 3: Module Installation**
Install session key modules, recovery modules, or other plugins during or after deployment.

## Gotchas

**Gas Estimation**: Always overestimate gas limits initially. Bundlers may reject operations with insufficient gas.

**Nonce Management**: Each smart wallet maintains nonces per key. Use key=0 for the default nonce sequence.

**Signature Validation**: Smart wallets validate signatures differently than EOAs. Ensure your signature format matches the wallet implementation (EIP-1271).

**Factory Initialization**: Some factories require initialization parameters. Check factory documentation for required constructor args.

**Bundler Selection**: Different bundlers have different policies. Some require minimum balance, others whitelist contracts.