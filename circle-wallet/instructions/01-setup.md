# Circle Wallet Setup Guide

## Prerequisites

- Circle Developer Account (https://console.circle.com)
- API Key from Circle Console
- Node.js 18+ or Python 3.8+

## Installation

### Node.js SDK
```bash
npm install @circle-fin/user-controlled-wallets @circle-fin/developer-controlled-wallets
```

## Configuration

### API Key Setup
```javascript
const { initiateDeveloperControlledWalletsClient } = require('@circle-fin/developer-controlled-wallets');

const client = initiateDeveloperControlledWalletsClient({
  apiKey: process.env.CIRCLE_API_KEY,
  entitySecret: process.env.CIRCLE_ENTITY_SECRET
});
```

## Creating a Wallet on Base

### Step 1: Create a Wallet Set
```javascript
const walletSetResponse = await client.createWalletSet({
  idempotencyKey: crypto.randomUUID(),
  name: 'My Base Wallets'
});
```

### Step 2: Create a Wallet
```javascript
const walletResponse = await client.createWallets({
  idempotencyKey: crypto.randomUUID(),
  blockchains: ['BASE'],
  count: 1,
  walletSetId: walletSetResponse.data.walletSet.id
});
```

## Key Contract Addresses (Base)
- **USDC**: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`

## Rate Limits
- Wallet creation: 10 requests per second
- Transaction submission: 50 requests per second
- Balance queries: 100 requests per second

## Error Handling
- Always use idempotency keys to prevent duplicate operations
- Handle 429 (rate limit) responses with exponential backoff
- Verify transaction status asynchronously via webhooks or polling
