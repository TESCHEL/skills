# Server Implementation

This guide covers implementing x402 payment protection on your server using the x402 Server SDK. The server validates incoming payments, delegates settlement to the on-chain facilitator contract, and enforces access control for paid resources.

## Quick Start with Express.js

Install the required packages:

```bash
npm install @x402/server @x402/evm express
```

Add payment protection to your Express routes:

```javascript
const express = require('express');
const { paymentMiddleware } = require('@x402/server');

const app = express();

app.use('/api/premium', paymentMiddleware({
  payTo: '0x1234567890123456789012345678901234567890',
  facilitator: '0x2345678901234567890123456789012345678901',
  chainId: 8453,
  rpcUrl: 'https://mainnet.base.org',
  routes: [{
    path: '/data',
    amount: '1000000',
    currency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
    description: 'Premium data access',
    methods: ['GET']
  }]
}));

app.get('/api/premium/data', (req, res) => {
  res.json({ data: 'Premium content', timestamp: Date.now() });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

The middleware automatically returns 402 responses for unpaid requests and validates payment signatures for paid requests.

## paymentMiddleware Configuration Reference

The `paymentMiddleware` function accepts a configuration object with the following properties:

**Required Properties:**

- `payTo` (string): Ethereum address receiving the payment. This is typically your service's payment collection address.

- `facilitator` (string): Address of the deployed x402 facilitator contract. The facilitator handles atomic settlement and prevents double-spending. Use the facilitator address for your target chain.

- `chainId` (number): EIP-155 chain ID where payments are settled (8453 for Base mainnet, 84532 for Base Sepolia testnet).

- `rpcUrl` (string): RPC endpoint for the blockchain. For Base mainnet, use `https://mainnet.base.org`.

- `routes` (array): Array of route configuration objects defining payment requirements. Each route object contains:
  - `path` (string): URL path pattern (supports Express route patterns like `/users/:id`)
  - `amount` (string): Payment amount in token base units (e.g., '1000000' for 1 USDC with 6 decimals)
  - `currency` (string): ERC-20 token contract address (e.g., '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913' for USDC on Base)
  - `description` (string): Human-readable description shown to users during payment
  - `methods` (array): HTTP methods requiring payment (e.g., ['GET', 'POST'])

**Optional Properties:**

- `acceptedMethods` (array): List of accepted payment method identifiers. Defaults to ['eip-4337', 'eoa']. Method identifiers reference registered implementations in the ERC-8004 registry.

- `expirySeconds` (number): Payment signature validity period in seconds. Defaults to 300 (5 minutes). After expiry, clients must request a new payment challenge.

- `onPaymentSettled` (function): Callback invoked after successful payment settlement. Receives `(paymentDetails, req, res)` arguments. Use this for logging, webhooks, or triggering business logic.

- `onPaymentFailed` (function): Callback invoked when payment verification or settlement fails. Receives `(error, req, res)` arguments. Useful for monitoring fraud attempts or debugging.

Example with all properties:

```javascript
app.use('/api/premium', paymentMiddleware({
  payTo: '0x1234567890123456789012345678901234567890',
  facilitator: '0x2345678901234567890123456789012345678901',
  chainId: 8453,
  rpcUrl: 'https://mainnet.base.org',
  routes: [{
    path: '/data',
    amount: '1000000',
    currency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
    description: 'Premium data access',
    methods: ['GET']
  }],
  acceptedMethods: ['eip-4337', 'eoa'],
  expirySeconds: 600,
  onPaymentSettled: (details, req, res) => {
    console.log('Payment settled:', details.txHash);
  },
  onPaymentFailed: (error, req, res) => {
    console.error('Payment failed:', error.message);
  }
}));
```

## How the Middleware Works

### Request Without Payment (402 Flow)

When a client makes a request without a valid payment signature:

1. Middleware checks the request path and method against configured routes
2. If payment is required, middleware generates a unique payment challenge (nonce)
3. Returns HTTP 402 with `PAYMENT-REQUIRED` header containing JSON payload:

```json
{
  "payTo": "0x1234567890123456789012345678901234567890",
  "amount": "1000000",
  "currency": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "chainId": 8453,
  "nonce": "0x3456789012345678901234567890123456789012345678901234567890123456",
  "description": "Premium data access",
  "acceptedMethods": ["eip-4337", "eoa"]
}
```

4. Client receives 402 response and initiates payment flow (see [Smart Wallet Skills](01-smart-wallet-skills.md))

### Request With Payment (Verify and Settle Flow)

When a client includes a payment signature via the `PAYMENT-SIGNATURE` header:

1. Middleware extracts and decodes the payment signature
2. Validates signature parameters match route requirements (amount, currency, payTo)
3. Checks signature expiry timestamp
4. Delegates settlement to the facilitator contract by calling `facilitator.settlePayment()`
5. Facilitator contract atomically verifies the payment method and transfers tokens
6. On successful settlement:
   - Invokes `onPaymentSettled` callback if configured
   - Allows request to proceed to route handler
7. On settlement failure:
   - Invokes `onPaymentFailed` callback if configured
   - Returns HTTP 402 with error details

The settlement delegation ensures atomic execution and prevents double-spending attacks.

## Dynamic Pricing

For endpoints with variable pricing based on request parameters, use the low-level `createPaymentRequired` and `verifyAndSettle` functions instead of the middleware:

```javascript
const { createPaymentRequired, verifyAndSettle } = require('@x402/server');

app.get('/api/reports/:reportId', async (req, res) => {
  const reportId = req.params.reportId;
  
  // Calculate price based on report complexity
  const report = await getReportMetadata(reportId);
  const amount = calculatePrice(report.pages, report.dataPoints);
  
  const paymentSignature = req.headers['payment-signature'];
  
  if (!paymentSignature) {
    const paymentRequired = createPaymentRequired({
      payTo: '0x1234567890123456789012345678901234567890',
      amount: amount.toString(),
      currency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
      chainId: 8453,
      description: `Report ${reportId} (${report.pages} pages)`,
      acceptedMethods: ['eip-4337', 'eoa'],
      expirySeconds: 300
    });
    
    res.setHeader('PAYMENT-REQUIRED', JSON.stringify(paymentRequired));
    return res.status(402).json({ error: 'Payment required' });
  }
  
  try {
    const settlement = await verifyAndSettle({
      paymentSignature,
      expectedPayTo: '0x1234567890123456789012345678901234567890',
      expectedAmount: amount.toString(),
      expectedCurrency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
      facilitator: '0x2345678901234567890123456789012345678901',
      chainId: 8453,
      rpcUrl: 'https://mainnet.base.org'
    });
    
    console.log('Payment settled:', settlement.txHash);
    const reportData = await generateReport(reportId);
    res.json(reportData);
  } catch (error) {
    res.status(402).json({ error: error.message });
  }
});
```

This approach gives you full control over pricing logic and payment verification.

## Server SDK API Reference

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `paymentMiddleware` | `config: PaymentConfig` | Express middleware | Creates middleware for automatic payment enforcement |
| `createPaymentRequired` | `params: PaymentParams` | `PaymentRequired` | Generates payment challenge for 402 responses |
| `verifyAndSettle` | `options: VerifyOptions` | `Promise<Settlement>` | Verifies signature and settles payment via facilitator |
| `decodePaymentSignature` | `signature: string` | `DecodedPayment` | Decodes base64-encoded payment signature |
| `createPaymentResponse` | `txHash: string, chainId: number` | `string` | Creates response header value after settlement |

**Type Definitions:**

```typescript
interface PaymentConfig {
  payTo: string;
  facilitator: string;
  chainId: number;
  rpcUrl: string;
  routes: RouteConfig[];
  acceptedMethods?: string[];
  expirySeconds?: number;
  onPaymentSettled?: (details: Settlement, req: Request, res: Response) => void;
  onPaymentFailed?: (error: Error, req: Request, res: Response) => void;
}

interface RouteConfig {
  path: string;
  amount: string;
  currency: string;
  description: string;
  methods: string[];
}

interface PaymentParams {
  payTo: string;
  amount: string;
  currency: string;
  chainId: number;
  description: string;
  acceptedMethods: string[];
  expirySeconds: number;
}

interface VerifyOptions {
  paymentSignature: string;
  expectedPayTo: string;
  expectedAmount: string;
  expectedCurrency: string;
  facilitator: string;
  chainId: number;
  rpcUrl: string;
}

interface Settlement {
  txHash: string;
  blockNumber: number;
  from: string;
  to: string;
  amount: string;
  currency: string;
}
```

## Settlement Verification and Trustless Architecture

The x402 protocol delegates payment settlement to an on-chain facilitator contract rather than allowing servers to directly verify payments. This architecture provides critical security guarantees:

**Atomic Settlement:** The facilitator contract executes payment verification and token transfer in a single transaction. Either both operations succeed or both revert. This prevents scenarios where a payment is verified but tokens are not transferred.

**Double-Spend Prevention:** The facilitator maintains nonce tracking for each payment method. Once a payment signature is used for settlement, the nonce is marked as consumed on-chain. Subsequent attempts to reuse the same signature will fail at the contract level, regardless of how many servers receive the signature.

**Trustless Verification:** Servers do not need to trust each other or maintain synchronized state. The blockchain serves as the single source of truth for payment status. Multiple servers can independently query the facilitator contract to verify payment without coordination.

**Payment Method Abstraction:** The facilitator delegates to payment method implementations registered in the ERC-8004 registry (see [ERC-8004 Registry Skills](02-erc-8004-registry.md)). This allows the protocol to support EOA signatures, EIP-4337 user operations, smart wallet signatures, and future payment methods without server-side changes.

**Fraud Mitigation:** Because settlement occurs on-chain, attempted fraud (invalid signatures, insufficient balances, unauthorized tokens) is detected by the facilitator contract. Servers receive cryptographic proof of settlement via transaction receipts.

The server's role is limited to:
1. Generating payment challenges (nonces)
2. Validating that payment parameters match route requirements
3. Submitting settlement transactions to the facilitator
4. Enforcing access control based on settlement results

This separation of concerns creates a trustless architecture where servers cannot steal funds, clients cannot double-spend, and payment verification is cryptographically secured.

## Webhook and Event Monitoring

The `onPaymentSettled` callback provides a hook for integrating with external systems after successful payment:

```javascript
app.use('/api/premium', paymentMiddleware({
  payTo: '0x1234567890123456789012345678901234567890',
  facilitator: '0x2345678901234567890123456789012345678901',
  chainId: 8453,
  rpcUrl: 'https://mainnet.base.org',
  routes: [{
    path: '/data',
    amount: '1000000',
    currency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
    description: 'Premium data access',
    methods: ['GET']
  }],
  onPaymentSettled: async (settlement, req, res) => {
    // Log to database
    await db.payments.insert({
      txHash: settlement.txHash,
      from: settlement.from,
      amount: settlement.amount,
      timestamp: Date.now(),
      path: req.path
    });
    
    // Send webhook to billing system
    await fetch('https://billing.example.com/webhook', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        event: 'payment.settled',
        txHash: settlement.txHash,
        customer: settlement.from,
        amount: settlement.amount
      })
    });
    
    // Update metrics
    metrics.recordPayment(settlement.amount);
  }
}));
```

The callback receives the full settlement details including transaction hash, block number, and token amounts. Use this for analytics, accounting, or triggering downstream workflows.

## Testing with Base Sepolia

For development and testing, use Base Sepolia testnet:

```javascript
app.use('/api/premium', paymentMiddleware({
  payTo: '0x1234567890123456789012345678901234567890',
  facilitator: '0x3456789012345678901234567890123456789012',
  chainId: 84532,
  rpcUrl: 'https://sepolia.base.org',
  routes: [{
    path: '/data',
    amount: '1000000',
    currency: '0x036CbD53842c5426634e7929541eC2318f3dCF7e',
    description: 'Test data access',
    methods: ['GET']
  }]
}));
```

Note the different facilitator contract address and USDC token address for Sepolia. Obtain Sepolia ETH from faucets and test USDC from the Base Sepolia USDC faucet.

Always test payment flows end-to-end on testnet before deploying to mainnet. Verify that:
- 402 responses include correct payment parameters
- Client wallets can sign and submit payments
- Settlement transactions confirm on-chain
- Subsequent requests with valid signatures succeed

## CORS Configuration for Browser Clients

When serving browser-based clients, configure CORS to expose x402 protocol headers:

```javascript
const cors = require('cors');

app.use(cors({
  origin: 'https://app.example.com',
  exposedHeaders: ['PAYMENT-REQUIRED', 'PAYMENT-RESPONSE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'PAYMENT-SIGNATURE'],
  credentials: true
}));

app.use('/api/premium', paymentMiddleware({
  // ... configuration
}));
```

**Key CORS settings:**

- `exposedHeaders`: Must include `PAYMENT-REQUIRED` (for 402 responses) and `PAYMENT-RESPONSE` (for settlement confirmations). Without these, browsers will not expose the headers to JavaScript.

- `allowedHeaders`: Must include `PAYMENT-SIGNATURE` to allow clients to send payment signatures in request headers.

- `credentials`: Set to `true` if using authentication cookies alongside payments.

Without proper CORS configuration, browser clients cannot read payment challenges or send payment signatures, breaking the payment flow.

## Production Configuration Example

Complete production server setup for Base mainnet:

```javascript
const express = require('express');
const cors = require('cors');
const { paymentMiddleware } = require('@x402/server');

const app = express();

app.use(cors({
  origin: process.env.CLIENT_ORIGIN,
  exposedHeaders: ['PAYMENT-REQUIRED', 'PAYMENT-RESPONSE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'PAYMENT-SIGNATURE'],
  credentials: true
}));

app.use('/api/premium', paymentMiddleware({
  payTo: process.env.PAYMENT_ADDRESS,
  facilitator: process.env.FACILITATOR_ADDRESS,
  chainId: 8453,
  rpcUrl: 'https://mainnet.base.org',
  routes: [
    {
      path: '/data',
      amount: '1000000',
      currency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
      description: 'Premium data access',
      methods: ['GET']
    },
    {
      path: '/reports/:id',
      amount: '5000000',
      currency: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
      description: 'Detailed report generation',
      methods: ['POST']
    }
  ],
  expirySeconds: 600,
  onPaymentSettled: async (settlement, req, res) => {
    await logPayment(settlement);
    await notifyBillingSystem(settlement);
  },
  onPaymentFailed: (error, req, res) => {
    console.error('Payment failed:', error);
    monitoringService.recordFailure(error);
  }
}));

app.get('/api/premium/data', (req, res) => {
  res.json({ data: 'Premium content' });
});

app.post('/api/premium/reports/:id', async (req, res) => {
  const report = await generateReport(req.params.id);
  res.json(report);
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

This configuration protects premium endpoints with USDC payments on Base mainnet while providing proper CORS support and monitoring hooks.

## Next Steps

- Review [Smart Wallet Skills](01-smart-wallet-skills.md) for client-side payment implementation
- Understand [ERC-8004 Registry Skills](02-erc-8004-registry.md) for payment method registration
- Deploy and test on Base Sepolia before mainnet deployment
- Monitor settlement transactions and payment metrics
- Configure proper error handling and retry logic for RPC failures