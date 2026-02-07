# Gasless Transactions with Circle Wallets

Gasless transactions eliminate the need for users to hold native tokens (ETH) to pay transaction fees. This guide explains how to use paymasters with Circle Wallets on Base to sponsor gas fees for your users.

## What is a Paymaster?

A Paymaster is a third-party smart contract that sponsors gas fees for UserOperations in the ERC-4337 account abstraction framework. When a Paymaster sponsors a transaction, the user pays nothing -- the Paymaster covers all gas costs.

Paymasters enable:

- **Better UX**: Users don't need ETH to transact
- **Onboarding**: New users can interact immediately without acquiring gas tokens
- **Subsidized operations**: Apps can cover costs for specific user actions
- **Gasless token transfers**: Send USDC or other tokens without holding ETH

## Coinbase Paymaster on Base

Coinbase provides a production Paymaster service on Base with generous gas credits:

- **Endpoint**: https://developer.coinbase.com/paymaster/v1/base
- **Gas credits**: Up to $15,000 in sponsored gas fees
- **Network**: Base mainnet and Base Sepolia testnet
- **Policy**: Sponsors standard token transfers and DeFi operations

This Paymaster integrates seamlessly with Circle Wallets and other ERC-4337 smart accounts.

## How Paymaster Works with ERC-4337

The Paymaster flow follows these steps:

### 1. Construct UserOperation Without Gas Payment

Create a UserOperation for your desired transaction (e.g., USDC transfer). Initially, leave paymaster fields empty:

```javascript
const userOp = {
  sender: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  nonce: "0x01",
  initCode: "0x",
  callData: "0x...", // encoded transfer call
  callGasLimit: "0x0",
  verificationGasLimit: "0x0",
  preVerificationGas: "0x0",
  maxFeePerGas: "0x0",
  maxPriorityFeePerGas: "0x0",
  paymasterAndData: "0x", // empty initially
  signature: "0x"
};
```

### 2. Send to Paymaster for Sponsorship

Request sponsorship using two JSON-RPC methods:

**pm_getPaymasterStubData**: Get initial paymaster data for gas estimation

```javascript
const stubResponse = await fetch(paymasterUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "pm_getPaymasterStubData",
    params: [userOp, entryPointAddress, chainId]
  })
});
```

**pm_getPaymasterData**: Get final signed paymaster data after gas estimation

```javascript
const paymasterResponse = await fetch(paymasterUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 2,
    method: "pm_getPaymasterData",
    params: [userOp, entryPointAddress, chainId]
  })
});

const { paymasterAndData } = await paymasterResponse.json();
userOp.paymasterAndData = paymasterAndData;
```

### 3. Paymaster Signs the UserOp

The Paymaster validates the UserOperation against its policies (rate limits, allowed contracts, operation types). If approved, it returns `paymasterAndData` containing:

- Paymaster contract address (20 bytes)
- Validation data and signature proving the Paymaster agrees to pay gas

### 4. Submit to Bundler and EntryPoint

Send the sponsored UserOperation to a Bundler:

```javascript
const bundlerResponse = await fetch(bundlerUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 3,
    method: "eth_sendUserOperation",
    params: [userOp, entryPointAddress]
  })
});
```

The EntryPoint contract verifies the Paymaster signature on-chain, confirming the Paymaster will cover gas costs.

### 5. Paymaster Pays Gas

When the UserOperation executes:

- EntryPoint calls `validatePaymasterUserOp` on the Paymaster contract
- Paymaster validates and deposits gas payment to EntryPoint
- User's transaction executes
- User pays nothing -- Paymaster covers all gas fees

## Integration with Circle Wallets

Circle's API makes gasless transactions simple with the `paymasterUrl` field. The SDK handles all Paymaster interactions automatically.

### Gasless USDC Transfer via Circle API

Send USDC without requiring the user to hold ETH:

```javascript
const axios = require("axios");

const CIRCLE_API_KEY = process.env.CIRCLE_API_KEY;
const USER_TOKEN = process.env.USER_TOKEN;

const response = await axios.post(
  "https://api.circle.com/v1/w3s/user/transactions/transfer",
  {
    userId: "user-uuid-here",
    walletId: "wallet-uuid-here",
    blockchain: "BASE",
    tokenAddress: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC on Base
    destinationAddress: "0x456...",
    amounts: ["10.50"],
    paymasterUrl: "https://developer.coinbase.com/paymaster/v1/base"
  },
  {
    headers: {
      "Authorization": `Bearer ${CIRCLE_API_KEY}`,
      "X-User-Token": USER_TOKEN,
      "Content-Type": "application/json"
    }
  }
);

console.log("Transaction ID:", response.data.data.id);
console.log("Gas paid by Paymaster");
```

Circle's backend:

1. Constructs UserOperation for the USDC transfer
2. Requests sponsorship from Coinbase Paymaster
3. Submits sponsored UserOp to Bundler
4. Returns transaction ID to your application

The user pays no gas -- the Paymaster sponsors the entire transaction.

### Contract Execution with Paymaster

Execute arbitrary contract calls gaslessly:

```javascript
const executeResponse = await axios.post(
  "https://api.circle.com/v1/w3s/user/transactions/contractExecution",
  {
    userId: "user-uuid-here",
    walletId: "wallet-uuid-here",
    blockchain: "BASE",
    contractAddress: "0x789...",
    abiFunctionSignature: "swap(uint256,uint256)",
    abiParameters: ["1000000", "950000"],
    paymasterUrl: "https://developer.coinbase.com/paymaster/v1/base"
  },
  {
    headers: {
      "Authorization": `Bearer ${CIRCLE_API_KEY}`,
      "X-User-Token": USER_TOKEN,
      "Content-Type": "application/json"
    }
  }
);
```

## Using Viem with Coinbase Paymaster

For direct integration without Circle's API, use viem's account abstraction support:

```javascript
import { createSmartAccountClient } from "viem/account-abstraction";
import { http, encodeFunctionData, parseEther } from "viem";
import { base } from "viem/chains";
import { toSimpleSmartAccount } from "viem/accounts";

// Create paymaster transport
const paymasterTransport = http(
  "https://developer.coinbase.com/paymaster/v1/base"
);

// Create smart account client with paymaster
const client = createSmartAccountClient({
  account: await toSimpleSmartAccount({
    client: publicClient,
    owner: privateKeyAccount,
    entryPoint: "0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789"
  }),
  chain: base,
  bundlerTransport: http("https://bundler.example.com/rpc"),
  paymasterTransport: paymasterTransport
});

// Send gasless UserOperation
const txHash = await client.sendTransaction({
  to: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC
  data: encodeFunctionData({
    abi: erc20ABI,
    functionName: "transfer",
    args: ["0x456...", parseEther("10")]
  }),
  value: 0n
});

console.log("Gasless transaction hash:", txHash);
```

Viem automatically:

- Calls `pm_getPaymasterStubData` for gas estimation
- Calls `pm_getPaymasterData` for final sponsorship
- Includes `paymasterAndData` in the UserOperation
- Submits to the Bundler

## Paymaster Policies

Paymasters enforce policies to control which operations they sponsor:

### Operation Allowlisting

Coinbase Paymaster sponsors:

- ERC-20 token transfers (USDC, USDT, DAI, etc.)
- Stablecoin swaps on major DEXs
- Common DeFi operations (Uniswap, Aave, Compound)
- NFT marketplace interactions

Blocked operations:

- High-risk contract calls
- Unverified contracts
- Operations exceeding gas limits

### Rate Limiting

Paymasters implement rate limits per account:

- Maximum transactions per hour per wallet
- Daily spending caps per account
- Network-wide rate limiting during congestion

### Contract Allowlists

Some Paymasters only sponsor calls to approved contracts:

```javascript
const allowedContracts = [
  "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC
  "0x4200000000000000000000000000000000000006", // WETH
  "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"  // Another token
];
```

The Paymaster rejects UserOperations targeting other addresses.

## Gas Estimation with Paymaster

Accurate gas estimation requires including the Paymaster:

```javascript
// Step 1: Get paymaster stub data for estimation
const stubResponse = await fetch(paymasterUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "pm_getPaymasterStubData",
    params: [
      userOp,
      "0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789", // EntryPoint
      "0x2105" // Base chain ID
    ]
  })
});

const { paymasterAndData: stubData } = await stubResponse.json();
userOp.paymasterAndData = stubData;

// Step 2: Estimate gas with stub data
const gasEstimate = await fetch(bundlerUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 2,
    method: "eth_estimateUserOperationGas",
    params: [
      userOp,
      "0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789"
    ]
  })
});

const gasLimits = await gasEstimate.json();
userOp.callGasLimit = gasLimits.callGasLimit;
userOp.verificationGasLimit = gasLimits.verificationGasLimit;
userOp.preVerificationGas = gasLimits.preVerificationGas;

// Step 3: Get final paymaster data with estimated gas
const finalPaymasterResponse = await fetch(paymasterUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 3,
    method: "pm_getPaymasterData",
    params: [
      userOp,
      "0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789",
      "0x2105"
    ]
  })
});

const { paymasterAndData: finalData } = await finalPaymasterResponse.json();
userOp.paymasterAndData = finalData;
```

Without stub data, gas estimation may be inaccurate because Paymaster validation adds overhead.

## Error Handling

Handle Paymaster failures gracefully:

### Paymaster Rejection

```javascript
try {
  const paymasterResponse = await fetch(paymasterUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0",
      id: 1,
      method: "pm_getPaymasterData",
      params: [userOp, entryPointAddress, chainId]
    })
  });

  const result = await paymasterResponse.json();

  if (result.error) {
    console.error("Paymaster rejected:", result.error.message);
    // Fallback: remove paymaster, user pays gas
    userOp.paymasterAndData = "0x";
    console.log("Falling back to self-pay transaction");
  } else {
    userOp.paymasterAndData = result.result.paymasterAndData;
  }
} catch (error) {
  console.error("Paymaster request failed:", error);
  // Fallback: proceed without paymaster
  userOp.paymasterAndData = "0x";
}
```

Common rejection reasons:

- **Policy violation**: Operation not allowed by Paymaster rules
- **Rate limit exceeded**: User has hit per-account or network-wide limits
- **Credit exhaustion**: Paymaster has depleted its gas deposit in EntryPoint
- **Invalid UserOp**: Malformed or suspicious UserOperation

### Fallback to Self-Pay

If Paymaster sponsorship fails, fall back to user-paid gas:

```javascript
function handlePaymasterFailure(userOp, walletBalance) {
  if (walletBalance < estimatedGasCost) {
    throw new Error(
      "Paymaster unavailable and insufficient ETH for gas. " +
      "Please add ETH to your wallet or try again later."
    );
  }

  // Remove paymaster data
  userOp.paymasterAndData = "0x";
  console.log("User will pay gas fees");

  return userOp;
}
```

Always check wallet ETH balance before attempting self-pay fallback.

## Testing on Base Sepolia

Test gasless transactions on Base Sepolia testnet:

```javascript
const SEPOLIA_PAYMASTER_URL = 
  "https://developer.coinbase.com/paymaster/v1/base-sepolia";

const testResponse = await axios.post(
  "https://api.circle.com/v1/w3s/user/transactions/transfer",
  {
    userId: "test-user-uuid",
    walletId: "test-wallet-uuid",
    blockchain: "BASE-SEPOLIA",
    tokenAddress: "0x036CbD53842c5426634e7929541eC2318f3dCF7e", // Test USDC
    destinationAddress: "0x456...",
    amounts: ["1.0"],
    paymasterUrl: SEPOLIA_PAYMASTER_URL
  },
  {
    headers: {
      "Authorization": `Bearer ${CIRCLE_API_KEY}`,
      "X-User-Token": USER_TOKEN,
      "Content-Type": "application/json"
    }
  }
);
```

Sepolia Paymaster provides unlimited test gas credits for development.

## Security Considerations

### Paymaster Validation

Paymasters must validate UserOperations to prevent abuse:

```solidity
function validatePaymasterUserOp(
    UserOperation calldata userOp,
    bytes32 userOpHash,
    uint256 maxCost
) external returns (bytes memory context, uint256 validationData) {
    // Verify sender is allowed
    require(allowedSenders[userOp.sender], "Sender not allowed");

    // Verify target contract is allowed
    address target = address(bytes20(userOp.callData[16:36]));
    require(allowedTargets[target], "Target not allowed");

    // Verify operation is within gas budget
    require(maxCost <= maxGasPerOp, "Gas cost too high");

    // Verify rate limit
    require(
        userOpCount[userOp.sender] < maxOpsPerDay,
        "Rate limit exceeded"
    );

    // Increment counter
    userOpCount[userOp.sender]++;

    return ("", 0); // Approve sponsorship
}
```

### Preventing Abuse

Mitigate Paymaster abuse:

- **Signature verification**: Ensure UserOps are signed by legitimate account owners
- **Nonce tracking**: Prevent replay attacks with proper nonce management
- **Gas limit caps**: Reject operations exceeding reasonable gas limits
- **Address allowlisting**: Only sponsor operations from verified smart accounts
- **Time-based limits**: Implement cooldowns between sponsored operations

### EntryPoint Security

The EntryPoint verifies Paymaster signatures on-chain:

```solidity
// EntryPoint validates paymaster signature
address paymaster = address(bytes20(paymasterAndData[0:20]));
require(
    paymaster.code.length > 0,
    "Paymaster must be a contract"
);

// Call paymaster validation
(bool success, bytes memory result) = paymaster.call(
    abi.encodeWithSelector(
        IPaymaster.validatePaymasterUserOp.selector,
        userOp,
        userOpHash,
        maxCost
    )
);

require(success, "Paymaster validation failed");
```

This prevents unauthorized gas sponsorship and ensures Paymasters explicitly approve each UserOperation.

## Next Steps

- **Review smart-wallet**: Understand Circle Wallet architecture and ERC-4337 integration
- **Explore x402**: Learn about bundlers, EntryPoints, and UserOperation lifecycle
- **Implement fallback logic**: Handle Paymaster failures gracefully in production
- **Monitor gas usage**: Track Paymaster credits and set up alerts for exhaustion
- **Test policies**: Verify your application's transactions comply with Paymaster rules

Gasless transactions dramatically improve user experience by eliminating the need for native gas tokens. With Circle Wallets and Coinbase Paymaster on Base, you can provide seamless onboarding and frictionless transactions for your users.