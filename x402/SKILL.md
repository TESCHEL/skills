# x402: HTTP 402 Payment Protocol

## Overview

x402 is an open standard for peer-to-peer HTTP-based payments using the HTTP 402 Payment Required status code. It enables seamless integration of blockchain payments into web services without requiring complex infrastructure or intermediaries.

## Protocol Flow

### Step 1: Initial Request

Client sends a standard HTTP request to a protected resource:

```bash
GET /api/protected-resource HTTP/1.1
Host: example.com
```

### Step 2: Payment Required Response

Server responds with HTTP 402 and payment details in the `PAYMENT-REQUIRED` header:

```http
HTTP/1.1 402 Payment Required
PAYMENT-REQUIRED: eyJhbW91bnQiOiIxMDAwMDAwIiwiY3VycmVuY3kiOiJVU0RDIiwiYWNjZXB0ZWRNZXRob2RzIjpbIkVJUC0zMDA5IiwiUGVybWl0MiJdLCJleHBpcnkiOjE3MzAwMDAwMDAsInBheW1lbnRJZCI6InV1aWQtMTIzNCJ9
```

The header contains base64-encoded JSON:

```json
{
 "amount": "1000000",
 "currency": "USDC",
 "acceptedMethods": ["EIP-3009", "Permit2"],
 "expiry": 1730000000,
 "paymentId": "uuid-1234"
}
```

### Step 3: Client Payment Construction

Client decodes payment details and constructs a `PaymentPayload` using EIP-3009 or Permit2:

```javascript
import { createPayment } from '@x402/evm';

const paymentPayload = await createPayment({
 amount: "1000000",
 asset: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC on Base
 payTo: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb1", // Merchant address
 maxTimeoutSeconds: 3600,
 authorization: {
 from: "0xUserWalletAddress",
 to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb1",
 value: "1000000",
 validAfter: 0,
 validBefore: Math.floor(Date.now() / 1000) + 3600,
 nonce: "0x1234567890abcdef",
 signature: "0x..."
 }
});
```

### Step 4: Retry with Payment Signature

Client re-sends the original request with the `PAYMENT-SIGNATURE` header:

```bash
GET /api/protected-resource HTTP/1.1
Host: example.com
PAYMENT-SIGNATURE: serialized_payload_and_signature_base64
```

### Step 5: Server Verification and Settlement

Server verifies the payment through the Treasure x402 facilitator contract and returns the protected content:

```http
HTTP/1.1 200 OK
PAYMENT-RESPONSE: {"transactionHash":"0xabc123...","blockNumber":12345678}
Content-Type: application/json

{"data":"protected content"}
```

## Supported Assets on Base

| Token | Symbol | Address |
|-------|--------|----------|
| USD Coin | USDC | 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 |
| MAGIC | MAGIC | Contact Treasure for current deployment |
| SMOL | SMOL | Contact Treasure for current deployment |
| MIO | MIO | Contact Treasure for current deployment |
| AI Frens | Various | Contact respective projects for addresses |

## Client Implementation

### JavaScript/TypeScript

#### Using @x402/fetch

```javascript
import { x402Fetch } from '@x402/fetch';
import { ethers } from 'ethers';

const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

const response = await x402Fetch('https://api.example.com/protected', {
 method: 'GET',
 signer: signer,
 paymentOptions: {
 maxAmount: '1000000',
 preferredAsset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913'
 }
});

const data = await response.json();
```

#### Using @x402/evm

```javascript
import { X402Client } from '@x402/evm';
import { createWalletClient, custom } from 'viem';
import { base } from 'viem/chains';

const walletClient = createWalletClient({
 chain: base,
 transport: custom(window.ethereum)
});

const client = new X402Client({
 walletClient,
 facilitatorAddress: '0xFacilitatorContractAddress'
});

const payment = await client.createPayment({
 amount: '1000000',
 asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
 payTo: '0xMerchantAddress',
 paymentId: 'payment-uuid-1234'
});

const signature = await client.signPayment(payment);
```

### Python

```python
from x402 import X402Client
from web3 import Web3

w3 = Web3(Web3.HTTPProvider('https://mainnet.base.org'))
account = w3.eth.account.from_key('0xPrivateKey')

client = X402Client(
 w3=w3,
 account=account,
 facilitator_address='0xFacilitatorContractAddress'
)

response = client.get(
 'https://api.example.com/protected',
 max_amount='1000000',
 preferred_asset='0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913'
)

data = response.json()
```

### Go

```go
import (
 "github.com/x402/go-x402"
 "github.com/ethereum/go-ethereum/ethclient"
)

client, err := ethclient.Dial("https://mainnet.base.org")
if err != nil {
 panic(err)
}

x402Client := x402.NewClient(x402.ClientConfig{
 EthClient: client,
 PrivateKey: privateKey,
 FacilitatorAddress: "0xFacilitatorContractAddress",
})

resp, err := x402Client.Get("https://api.example.com/protected", x402.PaymentOptions{
 MaxAmount: "1000000",
 PreferredAsset: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
})
```

## Server Implementation

### Node.js/Express

```javascript
const express = require('express');
const { paymentMiddleware } = require('@x402/server');

const app = express();

app.use('/api/protected', paymentMiddleware({
 amount: '1000000',
 currency: 'USDC',
 assetAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
 merchantAddress: '0xYourMerchantAddress',
 facilitatorAddress: '0xTreasureFacilitatorAddress',
 rpcUrl: 'https://mainnet.base.org',
 acceptedMethods: ['EIP-3009', 'Permit2']
}));

app.get('/api/protected', (req, res) => {
 res.json({ data: 'This is protected content' });
});
```

### Custom Verification

```javascript
const { verifyPayment } = require('@x402/server');
const { ethers } = require('ethers');

app.get('/api/protected', async (req, res) => {
 const paymentSignature = req.headers['payment-signature'];
 
 if (!paymentSignature) {
 const paymentRequired = {
 amount: '1000000',
 currency: 'USDC',
 acceptedMethods: ['EIP-3009'],
 expiry: Math.floor(Date.now() / 1000) + 3600,
 paymentId: generateUUID()
 };
 
 res.status(402)
 .header('PAYMENT-REQUIRED', 
 Buffer.from(JSON.stringify(paymentRequired)).toString('base64')
 )
 .send('Payment Required');
 return;
 }
 
 const provider = new ethers.JsonRpcProvider('https://mainnet.base.org');
 const verification = await verifyPayment({
 paymentSignature,
 facilitatorAddress: '0xTreasureFacilitatorAddress',
 expectedAmount: '1000000',
 expectedAsset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
 provider
 });
 
 if (verification.valid) {
 res.header('PAYMENT-RESPONSE', JSON.stringify({
 transactionHash: verification.txHash,
 blockNumber: verification.blockNumber
 }));
 res.json({ data: 'Protected content' });
 } else {
 res.status(402).send('Payment verification failed');
 }
});
```

## Smart Contracts

### Treasure x402 Facilitator

The facilitator contract handles payment verification and settlement:

**Base Mainnet:** Check Treasure documentation for latest address
**Base Sepolia:** Check Treasure documentation for testnet address

### Key Functions

#### verifyAndExecute

```solidity
function verifyAndExecute(
 address token,
 address from,
 address to,
 uint256 value,
 uint256 validAfter,
 uint256 validBefore,
 bytes32 nonce,
 bytes memory signature
) external returns (bool)
```

Verifies EIP-3009 authorization and executes the transfer atomically. Prevents double-spend by tracking used nonces.

#### Payment Validation

The facilitator ensures:
- Signature is valid for the authorization
- Current timestamp is within validAfter and validBefore window
- Nonce has not been previously used
- Token contract supports transferWithAuthorization (EIP-3009)
- Sufficient balance and allowance exist

### EIP-3009 Authorization Structure

```solidity
struct Authorization {
 address from; // Token holder
 address to; // Payment recipient
 uint256 value; // Amount to transfer
 uint256 validAfter; // Unix timestamp after which transfer is valid
 uint256 validBefore; // Unix timestamp before which transfer is valid
 bytes32 nonce; // Unique identifier to prevent replay
}
```

### Integration with ERC-20 Tokens

Tokens must implement EIP-3009 `transferWithAuthorization`:

```solidity
function transferWithAuthorization(
 address from,
 address to,
 uint256 value,
 uint256 validAfter,
 uint256 validBefore,
 bytes32 nonce,
 uint8 v,
 bytes32 r,
 bytes32 s
) external
```

USDC on Base (0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913) natively supports this interface.

## Security Considerations

### Double-Spend Prevention

The facilitator contract tracks used nonces on-chain. Each authorization nonce can only be used once, preventing replay attacks.

### Timeout Protection

Payments include `validAfter` and `validBefore` timestamps. Set `maxTimeoutSeconds` appropriately:

```javascript
const authorization = {
 validAfter: 0,
 validBefore: Math.floor(Date.now() / 1000) + 3600, // 1 hour
 // ...
};
```

### Signature Verification

All signatures use EIP-712 structured data signing:

```javascript
const domain = {
 name: 'USD Coin',
 version: '2',
 chainId: 8453, // Base mainnet
 verifyingContract: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913'
};

const types = {
 TransferWithAuthorization: [
 { name: 'from', type: 'address' },
 { name: 'to', type: 'address' },
 { name: 'value', type: 'uint256' },
 { name: 'validAfter', type: 'uint256' },
 { name: 'validBefore', type: 'uint256' },
 { name: 'nonce', type: 'bytes32' }
 ]
};
```

### Amount Verification

Servers must verify that the authorized amount matches the expected payment:

```javascript
if (authorization.value !== expectedAmount) {
 throw new Error('Amount mismatch');
}
```

## Live Adopters

The x402 protocol is actively used in production by:

- **1Pay.ing** -- Payment gateway service
- **Bino** -- Content monetization platform
- **Ekai Labs** -- AI model marketplace
- **Genbase** -- Generative content platform
- **Kodo** -- Decentralized storage payments
- **Nuwa** -- Creator economy tools
- **Oops!402** -- Payment testing and development tools
- **Primer** -- Educational content payments
- **PayStabl** -- Stablecoin payment processor

## Related Skills

- **smart-wallet** -- Abstract account integration for gasless x402 payments
- **erc-8004-registry** -- On-chain service discovery for x402-enabled endpoints

## Reference Documentation

For detailed specifications:

- **instructions/01-protocol-spec.md** -- Complete protocol specification, header formats, and payload schemas
- **instructions/02-client-implementation.md** -- Client SDK usage, wallet integration, and payment flow implementation
- **instructions/03-server-implementation.md** -- Server middleware setup, verification logic, and settlement patterns
- **instructions/04-smart-contracts.md** -- Facilitator contract architecture, on-chain verification, and deployment addresses

## Best Practices

### Client-Side

1. **Cache Payment Signatures** -- Avoid re-signing for retries within the validity window
2. **Handle 402 Gracefully** -- Present clear payment UI when 402 is received
3. **Validate Payment Details** -- Check amount, currency, and recipient before signing
4. **Use Appropriate Timeouts** -- Balance security with user experience (recommended: 5-60 minutes)
5. **Implement Retry Logic** -- Handle temporary network failures during settlement

### Server-Side

1. **Verify All Parameters** -- Check amount, asset, recipient, and timestamps
2. **Use Idempotency Keys** -- Prevent duplicate settlements with payment IDs
3. **Monitor Facilitator** -- Track gas costs and settlement times
4. **Set Reasonable Expiry** -- Give users enough time to complete payment (recommended: 10-30 minutes)
5. **Log All Transactions** -- Maintain audit trail of payment attempts and settlements

## Testing

### Base Sepolia Testnet

Test your integration on Base Sepolia before mainnet deployment:

```javascript
const testConfig = {
 chainId: 84532,
 rpcUrl: 'https://sepolia.base.org',
 usdcAddress: '0xSepoliaUSDCAddress',
 facilitatorAddress: '0xSepoliaFacilitatorAddress'
};
```

Obtain testnet USDC from Base Sepolia faucets or use test token contracts.

### Local Development

Use tools like Hardhat or Foundry to deploy local facilitator instances:

```bash
npx hardhat node
npx hardhat run scripts/deploy-facilitator.js --network localhost
```

## Migration Guide

Moving from traditional payment systems to x402:

### From API Keys

**Before:**
```javascript
fetch('https://api.example.com/data', {
 headers: { 'X-API-Key': 'secret-key' }
});
```

**After:**
```javascript
import { x402Fetch } from '@x402/fetch';

x402Fetch('https://api.example.com/data', {
 signer: ethersWalletSigner
});
```

### From Stripe/Traditional Payments

**Benefits:**
- No monthly fees or percentage-based charges
- Instant settlement (on-chain finality)
- No chargebacks or fraud
- Global reach without currency conversion
- User privacy (no personal data required)

**Trade-offs:**
- Requires blockchain wallet
- Gas fees for on-chain settlement (minimal on Base)
- No fiat off-ramp (users pay in crypto)

## Conclusion

x402 provides a lightweight, standard protocol for HTTP-based blockchain payments. With one-line server integration and multiple client SDKs, developers can monetize APIs and content without complex payment infrastructure. The protocol leverages existing EIP-3009 standards and the Treasure facilitator contract to ensure secure, verifiable payments with minimal integration overhead.