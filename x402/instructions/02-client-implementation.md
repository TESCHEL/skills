# Client Implementation

This guide covers implementing x402 payment clients in JavaScript, Python, and Go. All examples use real addresses and production-ready code patterns.

## JavaScript Client

### Installation

Install the x402 JavaScript SDK packages:

```bash
npm install @x402/fetch @x402/evm
```

The `@x402/fetch` package provides a drop-in replacement for the standard `fetch` API, while `@x402/evm` offers low-level EVM utilities for manual payment flows.

### Basic Usage with fetchWithPayment

The simplest integration uses `fetchWithPayment` as a drop-in replacement for `fetch`. It requires a viem `walletClient` for signing payment authorizations:

```javascript
import { fetchWithPayment } from '@x402/fetch';
import { createWalletClient, http } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount('0x...');
const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http()
});

// Drop-in replacement for fetch
const response = await fetchWithPayment(
  'https://api.example.com/data',
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' }
  },
  { walletClient }
);

const data = await response.json();
console.log('Received data:', data);
```

### How fetchWithPayment Works

The `fetchWithPayment` function handles the entire x402 payment flow automatically:

1. **Initial Request**: Sends the original HTTP request to the endpoint
2. **Intercept 402**: Detects `402 Payment Required` response with `X-Payment-Required` header
3. **Decode Payment Info**: Extracts payment details (amount, token, recipient, nonce, validUntil)
4. **Sign EIP-3009**: Creates a `transferWithAuthorization` signature using the wallet client
5. **Encode Signature**: Formats signature as hex string with v, r, s components
6. **Re-send Request**: Retries original request with `X-Payment` header containing encoded signature
7. **Return Response**: Returns the successful response or throws on failure

This flow is transparent to the caller -- you write code as if making a standard HTTP request.

### Advanced Manual Flow

For advanced use cases requiring fine-grained control, use `@x402/evm` directly:

```javascript
import {
  decodePaymentRequired,
  buildPaymentPayload,
  signPaymentPayload,
  encodePaymentSignature
} from '@x402/evm';
import { createWalletClient, http } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

const account = privateKeyToAccount('0x...');
const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http()
});

// Make initial request
const initialResponse = await fetch('https://api.example.com/data');

if (initialResponse.status === 402) {
  // Decode payment requirement from header
  const paymentHeader = initialResponse.headers.get('X-Payment-Required');
  const paymentInfo = decodePaymentRequired(paymentHeader);
  
  console.log('Payment required:', paymentInfo);
  // { amount: '1000000', token: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
  //   recipient: '0x...', nonce: '0x...', validUntil: 1735689600 }
  
  // Check price before proceeding
  if (BigInt(paymentInfo.amount) > BigInt('5000000')) {
    throw new Error('Price exceeds budget');
  }
  
  // Build EIP-3009 typed data payload
  const payload = buildPaymentPayload(paymentInfo, {
    from: account.address,
    chainId: 8453
  });
  
  // Sign the payload using wallet client
  const signature = await signPaymentPayload(walletClient, payload);
  
  // Encode signature for HTTP header
  const encodedSignature = encodePaymentSignature(signature);
  
  // Retry request with payment
  const paidResponse = await fetch('https://api.example.com/data', {
    headers: {
      'X-Payment': encodedSignature
    }
  });
  
  const data = await paidResponse.json();
  console.log('Received data:', data);
}
```

### @x402/evm API Reference

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `decodePaymentRequired` | `headerValue: string` | `PaymentInfo` | Decodes base64-encoded X-Payment-Required header into structured payment info |
| `buildPaymentPayload` | `paymentInfo: PaymentInfo, options: { from: Address, chainId: number }` | `TypedData` | Builds EIP-712 typed data payload for transferWithAuthorization |
| `signPaymentPayload` | `walletClient: WalletClient, payload: TypedData` | `Signature` | Signs EIP-712 payload using viem wallet client, returns { v, r, s } |
| `encodePaymentSignature` | `signature: Signature` | `string` | Encodes signature as hex string for X-Payment header (0x concatenated v, r, s) |
| `validatePaymentInfo` | `paymentInfo: PaymentInfo` | `boolean` | Validates payment info structure and checks validUntil timestamp |
| `estimateGas` | `walletClient: WalletClient, paymentInfo: PaymentInfo` | `bigint` | Estimates gas cost for settlement transaction on-chain |

## Python Client

### Installation

Install the x402 Python SDK:

```bash
pip install x402
```

The package requires Python 3.8+ and includes dependencies for EIP-712 signing with eth_account.

### Basic Usage

The `fetch_with_payment` function provides automatic payment handling:

```python
from x402 import fetch_with_payment
from eth_account import Account
import os

private_key = os.environ['PRIVATE_KEY']
account = Account.from_key(private_key)

response = fetch_with_payment(
    'https://api.example.com/data',
    account=account,
    chain_id=8453
)

data = response.json()
print('Received data:', data)
```

### Manual Flow

For advanced control, use the low-level functions:

```python
from x402 import (
    decode_payment_required,
    build_payment_payload,
    sign_payment_payload,
    encode_payment_signature
)
from eth_account import Account
import requests
import os

account = Account.from_key(os.environ['PRIVATE_KEY'])

# Make initial request
response = requests.get('https://api.example.com/data')

if response.status_code == 402:
    # Decode payment requirement
    payment_header = response.headers.get('X-Payment-Required')
    payment_info = decode_payment_required(payment_header)
    
    print(f'Payment required: {payment_info["amount"]} tokens')
    
    # Check price limit
    if int(payment_info['amount']) > 5_000_000:
        raise ValueError('Price exceeds budget')
    
    # Build EIP-712 payload
    payload = build_payment_payload(
        payment_info,
        from_address=account.address,
        chain_id=8453
    )
    
    # Sign with eth_account
    signature = sign_payment_payload(account, payload)
    
    # Encode for header
    encoded_sig = encode_payment_signature(signature)
    
    # Retry with payment
    paid_response = requests.get(
        'https://api.example.com/data',
        headers={'X-Payment': encoded_sig}
    )
    
    data = paid_response.json()
    print('Received data:', data)
```

## Go Client

### Installation

Install the x402 Go module:

```bash
go get github.com/x402/x402-go
```

### Basic Usage

The Go client provides a clean API with automatic payment handling:

```go
package main

import (
    "context"
    "fmt"
    "log"
    
    "github.com/ethereum/go-ethereum/crypto"
    "github.com/x402/x402-go/client"
)

func main() {
    // Load private key
    privateKey, err := crypto.HexToECDSA("your-private-key")
    if err != nil {
        log.Fatal(err)
    }
    
    // Create x402 client
    x402Client := client.New(&client.Config{
        PrivateKey: privateKey,
        ChainID:    8453, // Base
    })
    
    // Make request with automatic payment
    resp, err := x402Client.FetchWithPayment(
        context.Background(),
        "https://api.example.com/data",
        nil, // options
    )
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    fmt.Printf("Status: %d\n", resp.StatusCode)
}
```

## Wallet Integration Patterns

### Existing Viem Wallet

Integrate with any viem-compatible wallet:

```javascript
import { createWalletClient, custom } from 'viem';
import { base } from 'viem/chains';
import { fetchWithPayment } from '@x402/fetch';

// Use browser wallet (MetaMask, etc.)
const walletClient = createWalletClient({
  chain: base,
  transport: custom(window.ethereum)
});

const response = await fetchWithPayment(
  'https://api.example.com/data',
  {},
  { walletClient }
);
```

### Smart Wallet Compatibility (ERC-4337)

x402 works with ERC-4337 smart wallets that support EIP-712 signature validation. The payment signature is verified on-chain using ERC-1271:

```javascript
import { createWalletClient, http } from 'viem';
import { base } from 'viem/chains';
import { fetchWithPayment } from '@x402/fetch';
import { toSmartAccount } from '@/skills/smart-wallet';

// Create smart wallet account
const smartAccount = await toSmartAccount({
  owner: privateKeyToAccount('0x...'),
  factoryAddress: '0x...',
  entryPoint: '0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789'
});

const walletClient = createWalletClient({
  account: smartAccount,
  chain: base,
  transport: http()
});

// Works seamlessly with smart wallets
const response = await fetchWithPayment(
  'https://api.example.com/data',
  {},
  { walletClient }
);
```

See the [smart-wallet skill](/skills/smart-wallet) for complete smart wallet integration details.

## Budget and Approval Patterns

### Price Checking

Always validate payment amounts before signing:

```javascript
import { fetchWithPayment } from '@x402/fetch';
import { decodePaymentRequired } from '@x402/evm';

const MAX_PAYMENT = BigInt('10000000'); // 10 USDC (6 decimals)

try {
  const response = await fetchWithPayment(
    'https://api.example.com/data',
    {},
    {
      walletClient,
      beforePayment: async (paymentInfo) => {
        if (BigInt(paymentInfo.amount) > MAX_PAYMENT) {
          throw new Error(`Price ${paymentInfo.amount} exceeds budget`);
        }
      }
    }
  );
} catch (error) {
  console.error('Payment rejected:', error.message);
}
```

### Max Budget Enforcement

Track cumulative spending across multiple requests:

```javascript
class BudgetTracker {
  constructor(maxBudget) {
    this.maxBudget = BigInt(maxBudget);
    this.spent = BigInt(0);
  }
  
  async checkAndRecord(amount) {
    const amountBigInt = BigInt(amount);
    if (this.spent + amountBigInt > this.maxBudget) {
      throw new Error('Budget exhausted');
    }
    this.spent += amountBigInt;
  }
  
  reset() {
    this.spent = BigInt(0);
  }
}

const budget = new BudgetTracker('50000000'); // 50 USDC

for (const endpoint of endpoints) {
  const response = await fetchWithPayment(
    endpoint,
    {},
    {
      walletClient,
      beforePayment: async (paymentInfo) => {
        await budget.checkAndRecord(paymentInfo.amount);
      }
    }
  );
}

console.log(`Total spent: ${budget.spent}`);
```

## Error Handling

| Error Condition | HTTP Status | Error Code | Description | Recovery |
|----------------|-------------|------------|-------------|----------|
| Missing X-Payment-Required header | 402 | `NO_PAYMENT_HEADER` | Server returned 402 without payment details | Check server x402 middleware configuration |
| Expired payment requirement | 402 | `PAYMENT_EXPIRED` | validUntil timestamp has passed | Retry from initial request to get fresh nonce |
| Insufficient token balance | 400 | `INSUFFICIENT_BALANCE` | Payer lacks USDC balance to cover amount | Top up USDC at 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 on Base |
| Invalid signature | 400 | `INVALID_SIGNATURE` | Signature verification failed on-chain | Check wallet address, chain ID, and nonce correctness |
| Settlement timeout | 500 | `SETTLEMENT_TIMEOUT` | On-chain settlement took too long | Server issue -- retry after delay or contact provider |
| Double-spend detected | 400 | `NONCE_ALREADY_USED` | Nonce was already consumed in another transaction | Fetch new payment requirement with fresh nonce |

Example error handling:

```javascript
import { fetchWithPayment } from '@x402/fetch';

try {
  const response = await fetchWithPayment(
    'https://api.example.com/data',
    {},
    { walletClient }
  );
  const data = await response.json();
} catch (error) {
  if (error.code === 'INSUFFICIENT_BALANCE') {
    console.error('Please top up USDC on Base chain');
    console.error('USDC address: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913');
  } else if (error.code === 'PAYMENT_EXPIRED') {
    console.log('Payment expired, retrying...');
    // Automatic retry will get fresh nonce
  } else if (error.code === 'NONCE_ALREADY_USED') {
    console.error('Nonce collision detected');
    // Must fetch new payment requirement
  } else {
    console.error('Payment failed:', error.message);
  }
}
```

## Registry Integration

Discover x402-enabled API endpoints using the ERC-8004 registry:

```javascript
import { fetchWithPayment } from '@x402/fetch';
import { lookupEndpoint } from '@/skills/erc-8004-registry';
import { createPublicClient, http } from 'viem';
import { base } from 'viem/chains';

const publicClient = createPublicClient({
  chain: base,
  transport: http()
});

// Discover endpoint from registry
const endpointInfo = await lookupEndpoint(
  publicClient,
  'api.example.com',
  '/weather'
);

if (endpointInfo.supportsX402) {
  const response = await fetchWithPayment(
    `https://${endpointInfo.domain}${endpointInfo.path}`,
    {},
    { walletClient }
  );
  console.log('Payment processed via x402');
}
```

See the [erc-8004-registry skill](/skills/erc-8004-registry) for complete registry integration patterns.

## Testing

Test x402 clients against local development servers:

```javascript
import { fetchWithPayment } from '@x402/fetch';
import { createWalletClient, http } from 'viem';
import { foundry } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

// Use local Anvil node for testing
const testAccount = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80'
);

const walletClient = createWalletClient({
  account: testAccount,
  chain: foundry,
  transport: http('http://127.0.0.1:8545')
});

// Test against local server
const response = await fetchWithPayment(
  'http://localhost:3000/test',
  {},
  { walletClient }
);

console.log('Test payment successful');
```

This comprehensive guide covers all major client implementation patterns for x402. Combine with server-side middleware from previous sections for complete end-to-end integration.