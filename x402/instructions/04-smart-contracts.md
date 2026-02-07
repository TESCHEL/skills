# Smart Contracts

The x402 protocol relies on smart contracts to execute payment settlements on-chain. The core component is the **Facilitator contract**, which acts as a trusted intermediary between merchants and customers.

## Facilitator Contract Overview

The Facilitator contract provides secure, atomic payment settlement by:

- **Verifying EIP-3009 authorizations**: Validates signatures and parameters before execution
- **Executing token transfers**: Calls `transferWithAuthorization` on USDC contracts
- **Preventing double-spend**: Tracks used nonces to ensure each authorization is used exactly once
- **Routing funds atomically**: Transfers complete in a single transaction or revert entirely
- **Emitting settlement events**: Provides on-chain audit trail for all payments

The Facilitator contract acts as an intermediary that never holds funds -- it verifies signatures, marks nonces as used, and forwards the transfer instruction to the USDC contract, which executes the actual transfer from customer to merchant.

## Deployment Addresses

| Network | Chain ID | Contract Name | Address |
|---------|----------|---------------|----------|
| Base Mainnet | 8453 | Treasure x402 Facilitator | TBD (pending deployment) |
| Base Sepolia | 84532 | Treasure x402 Facilitator | TBD (pending deployment) |

## Core Functions

The Facilitator contract exposes the following interface:

```solidity
interface ITreasureX402Facilitator {
    // Settle a payment using EIP-3009 authorization
    function settlePayment(
        address token,
        address from,
        address to,
        uint256 value,
        uint256 validAfter,
        uint256 validBefore,
        bytes32 nonce,
        bytes memory signature
    ) external returns (bytes32 txHash);

    // Check if a nonce has been used
    function isNonceUsed(
        address from,
        bytes32 nonce
    ) external view returns (bool);

    // Emitted when a payment is successfully settled
    event PaymentSettled(
        address indexed token,
        address indexed from,
        address indexed to,
        uint256 value,
        bytes32 nonce,
        bytes32 txHash
    );
}
```

### settlePayment

Executes a payment settlement using an EIP-3009 authorization:

- **token**: Address of the token contract (USDC)
- **from**: Customer's wallet address (authorization signer)
- **to**: Merchant's receiving address
- **value**: Amount in token's smallest unit (6 decimals for USDC)
- **validAfter**: Unix timestamp when authorization becomes valid
- **validBefore**: Unix timestamp when authorization expires
- **nonce**: Random 32-byte value to prevent replay
- **signature**: EIP-712 signature from customer

Returns the transaction hash of the settlement.

### isNonceUsed

Checks whether a specific nonce has been consumed:

- **from**: Wallet address that created the authorization
- **nonce**: The nonce to check

Returns `true` if the nonce has been used, `false` otherwise.

## Settlement Flow Diagram

The on-chain settlement process follows these steps:

```
1. Merchant calls settlePayment(token, from, to, value, validAfter, validBefore, nonce, sig)
   |
   v
2. Facilitator verifies nonce not used: require(!isNonceUsed(from, nonce))
   |
   v
3. Facilitator marks nonce as used: usedNonces[from][nonce] = true
   |
   v
4. Facilitator calls USDC.transferWithAuthorization(from, to, value, validAfter, validBefore, nonce, sig)
   |
   v
5. USDC contract verifies signature and executes transfer: from -> to
   |
   v
6. Facilitator emits PaymentSettled event
   |
   v
7. Transaction hash returned to caller
```

This flow ensures atomic settlement -- either all steps succeed or the entire transaction reverts.

## EIP-3009 transferWithAuthorization

The USDC contract on Base implements EIP-3009, enabling gasless token transfers via signed authorizations.

### USDC Contract Address

- **Base Mainnet**: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- **Base Sepolia**: `0x036CbD53842c5426634e7929541eC2318f3dCF7e`

### Interface Definition

```solidity
interface IERC3009 {
    function transferWithAuthorization(
        address from,
        address to,
        uint256 value,
        uint256 validAfter,
        uint256 validBefore,
        bytes32 nonce,
        bytes memory signature
    ) external;

    function authorizationState(
        address authorizer,
        bytes32 nonce
    ) external view returns (bool);
}
```

### EIP-712 Domain for USDC

When creating signatures, use this domain separator:

```javascript
const domain = {
  name: "USD Coin",
  version: "2",
  chainId: 8453, // Base Mainnet
  verifyingContract: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
};
```

For Base Sepolia, use chainId `84532` and the Sepolia USDC address.

The typed data structure for `TransferWithAuthorization`:

```javascript
const types = {
  TransferWithAuthorization: [
    { name: "from", type: "address" },
    { name: "to", type: "address" },
    { name: "value", type: "uint256" },
    { name: "validAfter", type: "uint256" },
    { name: "validBefore", type: "uint256" },
    { name: "nonce", type: "bytes32" }
  ]
};
```

## Nonce Management

Nonces in x402 are **random 32-byte values** rather than sequential integers. This design provides several benefits:

### Why Random Nonces?

1. **No ordering dependency**: Authorizations can be used in any order, enabling parallel processing
2. **Privacy**: Sequential nonces leak information about transaction volume
3. **Collision resistance**: 2^256 possible values make collisions astronomically unlikely
4. **Simplicity**: No need to track and increment counters

### Generating Nonces

In JavaScript/TypeScript:

```javascript
import { randomBytes } from "crypto";

function generateNonce() {
  return "0x" + randomBytes(32).toString("hex");
}

// Example: "0x8f3d..."
const nonce = generateNonce();
```

In Python:

```python
import os

def generate_nonce():
    return "0x" + os.urandom(32).hex()

# Example: "0x7a2c..."
nonce = generate_nonce()
```

### Checking Nonce Status

Before settling, verify a nonce hasn't been used:

```javascript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const client = createPublicClient({
  chain: base,
  transport: http()
});

const isUsed = await client.readContract({
  address: "0x...", // Facilitator address
  abi: facilitatorAbi,
  functionName: "isNonceUsed",
  args: [
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", // from
    "0x8f3d..." // nonce
  ]
});

if (isUsed) {
  throw new Error("Nonce already used");
}
```

## Permit2 Support

For tokens that don't implement EIP-3009 (like WETH, DAI on some chains), the Facilitator supports **Permit2** as an alternative authorization mechanism.

### settlePaymentPermit2 Interface

```solidity
function settlePaymentPermit2(
    address token,
    address from,
    address to,
    uint256 value,
    uint256 deadline,
    bytes memory signature
) external returns (bytes32 txHash);
```

Permit2 uses a different signature structure but provides similar gasless transfer capabilities.

### Permit2 Contract Address

Permit2 is deployed at the same address across all chains:

- **All networks**: `0x000000000022D473030F116dDEE9F6B43aC78BA3`

This includes Base Mainnet, Base Sepolia, Ethereum, and other EVM chains.

## Security Architecture

The x402 protocol implements multiple layers of security:

### Double-Spend Prevention

The Facilitator contract maintains a mapping of used nonces:

```solidity
mapping(address => mapping(bytes32 => bool)) private usedNonces;
```

Before processing any authorization, the contract:

1. Checks `require(!usedNonces[from][nonce], "Nonce already used")`
2. Marks the nonce as used: `usedNonces[from][nonce] = true`
3. Only then calls `transferWithAuthorization`

This ensures each authorization can be executed exactly once.

### Signature Verification Chain

Signature validation happens at two levels:

1. **Facilitator level**: Optionally validates the signature matches expected signer
2. **USDC contract level**: The authoritative validation using ECDSA recovery and EIP-712

The USDC contract performs:

```solidity
// Reconstruct EIP-712 hash
bytes32 digest = keccak256(abi.encodePacked(
    "\x19\x01",
    DOMAIN_SEPARATOR,
    keccak256(abi.encode(TYPEHASH, from, to, value, validAfter, validBefore, nonce))
));

// Recover signer from signature
address recoveredSigner = ecrecover(digest, v, r, s);

// Verify signer matches 'from' address
require(recoveredSigner == from, "Invalid signature");
```

### Attack Prevention

| Attack Vector | Prevention Mechanism |
|---------------|----------------------|
| Double-spend | Nonce tracking in Facilitator contract |
| Expired authorization | `validBefore` timestamp checked on-chain |
| Premature authorization | `validAfter` timestamp checked on-chain |
| Amount manipulation | Value included in signed message |
| Wrong recipient | Recipient address included in signed message |
| Cross-chain replay | Chain ID in EIP-712 domain separator |
| Forged signature | ECDSA signature verification in USDC contract |
| Front-running | Nonce system allows customer to invalidate authorization |

## Interacting with Facilitator via viem

### Reading Contract State

Check if a nonce has been used:

```javascript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const facilitatorAbi = [
  {
    name: "isNonceUsed",
    type: "function",
    stateMutability: "view",
    inputs: [
      { name: "from", type: "address" },
      { name: "nonce", type: "bytes32" }
    ],
    outputs: [{ type: "bool" }]
  }
];

const client = createPublicClient({
  chain: base,
  transport: http()
});

const isUsed = await client.readContract({
  address: "0x...", // Facilitator address
  abi: facilitatorAbi,
  functionName: "isNonceUsed",
  args: [customerAddress, nonce]
});
```

### Watching for Settlement Events

Monitor successful payments:

```javascript
const paymentSettledAbi = [
  {
    name: "PaymentSettled",
    type: "event",
    inputs: [
      { indexed: true, name: "token", type: "address" },
      { indexed: true, name: "from", type: "address" },
      { indexed: true, name: "to", type: "address" },
      { indexed: false, name: "value", type: "uint256" },
      { indexed: false, name: "nonce", type: "bytes32" },
      { indexed: false, name: "txHash", type: "bytes32" }
    ]
  }
];

const unwatch = client.watchContractEvent({
  address: "0x...", // Facilitator address
  abi: paymentSettledAbi,
  eventName: "PaymentSettled",
  args: {
    to: merchantAddress // Filter for payments to specific merchant
  },
  onLogs: (logs) => {
    logs.forEach((log) => {
      console.log("Payment received:", {
        from: log.args.from,
        amount: log.args.value,
        nonce: log.args.nonce,
        txHash: log.args.txHash
      });
    });
  }
});

// Stop watching
// unwatch();
```

### Settling a Payment

```javascript
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { base } from "viem/chains";

const account = privateKeyToAccount("0x...");
const client = createWalletClient({
  account,
  chain: base,
  transport: http()
});

const settleAbi = [
  {
    name: "settlePayment",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "token", type: "address" },
      { name: "from", type: "address" },
      { name: "to", type: "address" },
      { name: "value", type: "uint256" },
      { name: "validAfter", type: "uint256" },
      { name: "validBefore", type: "uint256" },
      { name: "nonce", type: "bytes32" },
      { name: "signature", type: "bytes" }
    ],
    outputs: [{ type: "bytes32" }]
  }
];

const txHash = await client.writeContract({
  address: "0x...", // Facilitator address
  abi: settleAbi,
  functionName: "settlePayment",
  args: [
    "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC on Base
    customerAddress,
    merchantAddress,
    10000000n, // 10 USDC (6 decimals)
    0n,
    Math.floor(Date.now() / 1000) + 3600, // Valid for 1 hour
    nonce,
    signature
  ]
});

console.log("Settlement transaction:", txHash);
```

## Example Custom Facilitator Contract

You can deploy your own Facilitator contract with custom logic:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface IERC3009 {
    function transferWithAuthorization(
        address from,
        address to,
        uint256 value,
        uint256 validAfter,
        uint256 validBefore,
        bytes32 nonce,
        bytes memory signature
    ) external;
}

contract CustomX402Facilitator is Ownable, ReentrancyGuard {
    mapping(address => mapping(bytes32 => bool)) public usedNonces;
    
    event PaymentSettled(
        address indexed token,
        address indexed from,
        address indexed to,
        uint256 value,
        bytes32 nonce,
        bytes32 txHash
    );
    
    constructor() Ownable(msg.sender) {}
    
    function settlePayment(
        address token,
        address from,
        address to,
        uint256 value,
        uint256 validAfter,
        uint256 validBefore,
        bytes32 nonce,
        bytes memory signature
    ) external nonReentrant returns (bytes32) {
        require(!usedNonces[from][nonce], "Nonce already used");
        
        // Mark nonce as used before external call
        usedNonces[from][nonce] = true;
        
        // Call USDC contract to execute transfer
        IERC3009(token).transferWithAuthorization(
            from,
            to,
            value,
            validAfter,
            validBefore,
            nonce,
            signature
        );
        
        // Generate transaction hash
        bytes32 txHash = keccak256(abi.encodePacked(
            token, from, to, value, nonce, block.timestamp
        ));
        
        emit PaymentSettled(token, from, to, value, nonce, txHash);
        
        return txHash;
    }
    
    function isNonceUsed(
        address from,
        bytes32 nonce
    ) external view returns (bool) {
        return usedNonces[from][nonce];
    }
}
```

This simplified example demonstrates the core pattern. Production contracts should include:

- Fee collection mechanisms
- Multi-token support validation
- Emergency pause functionality
- Upgradability patterns
- Gas optimization

## Integration with Smart Wallets

When customers use smart wallets (see [Smart Wallet Skills](02-smart-wallet.md)), the authorization flow differs slightly:

1. Smart wallet signs the EIP-3009 message using ERC-1271
2. Facilitator validates the signature using `isValidSignature(hash, signature)`
3. Settlement proceeds normally if signature is valid

Smart wallet addresses can be registered in the [ERC-8004 Registry](03-erc-8004-registry.md) to enable seamless payment discovery.

## Related Documentation

- [Smart Wallet Skills](02-smart-wallet.md) -- Understanding smart contract wallets and ERC-1271 signatures
- [ERC-8004 Registry](03-erc-8004-registry.md) -- On-chain merchant and customer registration
- [Payment Flow](../examples/payment-flow.md) -- Complete payment lifecycle including smart contract interactions

---

**Security Note**: Always verify contract addresses before interacting. Use established block explorers like Basescan to confirm deployment addresses and source code.