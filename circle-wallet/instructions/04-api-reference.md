# API Reference

This document provides comprehensive API reference for Circle Programmable Wallets on Base. All endpoints require authentication and use REST conventions with JSON request/response bodies.

## Base URL

All API requests should be made to:

```
https://api.circle.com/v1/w3s
```

## Authentication

Circle API uses Bearer token authentication. Include your API key in the Authorization header:

```bash
Authorization: Bearer YOUR_API_KEY
```

For developer-controlled wallet operations, you must also provide an `entitySecretCiphertext` in request bodies. This encrypted secret is obtained during SDK initialization and proves you control the entity.

## Rate Limits

API requests are rate-limited to prevent abuse:

- Standard tier: 100 requests per minute
- Premium tier: 1000 requests per minute

Rate limit headers are included in responses:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
```

When rate limited, you receive a 429 status code with retry information.

## Pagination

List endpoints support cursor-based pagination using `pageSize`, `pageBefore`, and `pageAfter` query parameters:

```
GET /wallets?pageSize=10&pageAfter=eyJpZCI6IjEyMyJ9
```

Response includes pagination tokens:

```json
{
  "data": [...],
  "pagination": {
    "nextPageToken": "eyJpZCI6IjQ1NiJ9",
    "prevPageToken": "eyJpZCI6Ijc4OSJ9"
  }
}
```

## Core Endpoints

### 1. Create Wallet Set

Wallet sets group related wallets together for organizational purposes.

**Endpoint:** `POST /developer/walletSets`

**Request:**

```bash
curl -X POST https://api.circle.com/v1/w3s/developer/walletSets \
  -H "Authorization: Bearer test_api_key_live_12345abcdef" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production Wallets"
  }'
```

**Request Body:**

```json
{
  "name": "Production Wallets"
}
```

**Response (200 OK):**

```json
{
  "data": {
    "walletSet": {
      "id": "01234567-89ab-cdef-0123-456789abcdef",
      "custodyType": "DEVELOPER",
      "name": "Production Wallets",
      "createDate": "2024-01-15T10:30:00Z",
      "updateDate": "2024-01-15T10:30:00Z"
    }
  }
}
```

### 2. Create Developer-Controlled Wallet

Create one or more wallets on Base. Supports both EOA (Externally Owned Account) and SCA (Smart Contract Account) types. For SCA wallets, refer to the [smart-wallet](../smart-wallet/overview.md) skill for advanced features.

**Endpoint:** `POST /developer/wallets`

**Request:**

```bash
curl -X POST https://api.circle.com/v1/w3s/developer/wallets \
  -H "Authorization: Bearer test_api_key_live_12345abcdef" \
  -H "Content-Type: application/json" \
  -d '{
    "blockchains": ["BASE"],
    "count": 2,
    "walletSetId": "01234567-89ab-cdef-0123-456789abcdef",
    "accountType": "SCA",
    "entitySecretCiphertext": "encrypted_secret_xyz789"
  }'
```

**Request Body:**

```json
{
  "blockchains": ["BASE"],
  "count": 2,
  "walletSetId": "01234567-89ab-cdef-0123-456789abcdef",
  "accountType": "SCA",
  "entitySecretCiphertext": "encrypted_secret_xyz789"
}
```

**Parameters:**

- `blockchains`: Array of blockchain networks (use "BASE" for Base)
- `count`: Number of wallets to create (1-10)
- `walletSetId`: ID from Create Wallet Set response
- `accountType`: "EOA" for standard wallets or "SCA" for smart contract wallets
- `entitySecretCiphertext`: Encrypted entity secret for authorization

**Response (201 Created):**

```json
{
  "data": {
    "wallets": [
      {
        "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "state": "LIVE",
        "walletSetId": "01234567-89ab-cdef-0123-456789abcdef",
        "custodyType": "DEVELOPER",
        "address": "0x1234567890abcdef1234567890abcdef12345678",
        "blockchain": "BASE",
        "accountType": "SCA",
        "createDate": "2024-01-15T10:35:00Z",
        "updateDate": "2024-01-15T10:35:00Z"
      },
      {
        "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
        "state": "LIVE",
        "walletSetId": "01234567-89ab-cdef-0123-456789abcdef",
        "custodyType": "DEVELOPER",
        "address": "0xabcdef1234567890abcdef1234567890abcdef12",
        "blockchain": "BASE",
        "accountType": "SCA",
        "createDate": "2024-01-15T10:35:00Z",
        "updateDate": "2024-01-15T10:35:00Z"
      }
    ]
  }
}
```

### 3. Get Wallet Balances

Retrieve token balances for a specific wallet. Returns native ETH balance and all ERC-20 token balances.

**Endpoint:** `GET /wallets/{id}/balances`

**Request:**

```bash
curl -X GET https://api.circle.com/v1/w3s/wallets/a1b2c3d4-e5f6-7890-abcd-ef1234567890/balances \
  -H "Authorization: Bearer test_api_key_live_12345abcdef"
```

**Response (200 OK):**

```json
{
  "data": {
    "tokenBalances": [
      {
        "token": {
          "id": "native-eth-base",
          "blockchain": "BASE",
          "name": "Ethereum",
          "symbol": "ETH",
          "decimals": 18,
          "isNative": true,
          "tokenAddress": null
        },
        "amount": "1250000000000000000",
        "updateDate": "2024-01-15T11:00:00Z"
      },
      {
        "token": {
          "id": "usdc-base",
          "blockchain": "BASE",
          "name": "USD Coin",
          "symbol": "USDC",
          "decimals": 6,
          "isNative": false,
          "tokenAddress": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
        },
        "amount": "5000000000",
        "updateDate": "2024-01-15T11:00:00Z"
      }
    ]
  }
}
```

**Note:** The USDC token address on Base is `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`. Amounts are returned as strings in smallest unit (wei for ETH, smallest denomination for tokens).

### 4. Transfer Tokens

Execute a token transfer from a developer-controlled wallet. Supports both native ETH and ERC-20 tokens. For cross-chain transfers using ERC-7683, see the [x402](../x402/overview.md) skill.

**Endpoint:** `POST /developer/transactions/transfer`

**Request:**

```bash
curl -X POST https://api.circle.com/v1/w3s/developer/transactions/transfer \
  -H "Authorization: Bearer test_api_key_live_12345abcdef" \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "tokenId": "usdc-base",
    "destinationAddress": "0x9876543210fedcba9876543210fedcba98765432",
    "amounts": ["1000000000"],
    "feeLevel": "MEDIUM",
    "entitySecretCiphertext": "encrypted_secret_xyz789"
  }'
```

**Request Body:**

```json
{
  "walletId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "tokenId": "usdc-base",
  "destinationAddress": "0x9876543210fedcba9876543210fedcba98765432",
  "amounts": ["1000000000"],
  "feeLevel": "MEDIUM",
  "entitySecretCiphertext": "encrypted_secret_xyz789"
}
```

**Parameters:**

- `walletId`: Source wallet ID
- `tokenId`: Token identifier or "native-eth-base" for ETH
- `destinationAddress`: Recipient address
- `amounts`: Array of amounts in smallest unit (e.g., 1000000000 = 1000 USDC)
- `feeLevel`: Gas fee priority -- "LOW", "MEDIUM", or "HIGH"
- `entitySecretCiphertext`: Encrypted entity secret for authorization

**Response (201 Created):**

```json
{
  "data": {
    "id": "tx-12345678-90ab-cdef-1234-567890abcdef",
    "state": "INITIATED",
    "walletId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "tokenId": "usdc-base",
    "destinationAddress": "0x9876543210fedcba9876543210fedcba98765432",
    "amounts": ["1000000000"],
    "blockchain": "BASE",
    "txHash": null,
    "createDate": "2024-01-15T11:15:00Z",
    "updateDate": "2024-01-15T11:15:00Z"
  }
}
```

### 5. Get Transaction Status

Poll transaction status to track execution progress. Transactions move through states: INITIATED -> PENDING -> CONFIRMED -> COMPLETE.

**Endpoint:** `GET /transactions/{id}`

**Request:**

```bash
curl -X GET https://api.circle.com/v1/w3s/transactions/tx-12345678-90ab-cdef-1234-567890abcdef \
  -H "Authorization: Bearer test_api_key_live_12345abcdef"
```

**Response (200 OK):**

```json
{
  "data": {
    "transaction": {
      "id": "tx-12345678-90ab-cdef-1234-567890abcdef",
      "state": "COMPLETE",
      "txHash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
      "walletId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "tokenId": "usdc-base",
      "destinationAddress": "0x9876543210fedcba9876543210fedcba98765432",
      "amounts": ["1000000000"],
      "blockchain": "BASE",
      "gasLimit": "100000",
      "gasPrice": "50000000",
      "actualFee": "0.0025",
      "createDate": "2024-01-15T11:15:00Z",
      "updateDate": "2024-01-15T11:16:30Z"
    }
  }
}
```

**Transaction States:**

- `INITIATED`: Transaction created, waiting to be broadcast
- `PENDING`: Transaction broadcast to network, waiting for confirmation
- `CONFIRMED`: Transaction confirmed on-chain
- `COMPLETE`: Transaction finalized with sufficient confirmations
- `FAILED`: Transaction failed (insufficient funds, reverted, etc.)

### 6. Estimate Transfer Fee

Estimate gas fees before executing a transfer. Helpful for displaying costs to users.

**Endpoint:** `POST /transactions/transfer/estimateFee`

**Request:**

```bash
curl -X POST https://api.circle.com/v1/w3s/transactions/transfer/estimateFee \
  -H "Authorization: Bearer test_api_key_live_12345abcdef" \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "tokenId": "usdc-base",
    "destinationAddress": "0x9876543210fedcba9876543210fedcba98765432",
    "amounts": ["1000000000"],
    "feeLevel": "MEDIUM"
  }'
```

**Response (200 OK):**

```json
{
  "data": {
    "low": {
      "gasLimit": "100000",
      "gasPrice": "30000000",
      "maxFee": "0.003",
      "baseFee": "0.0025",
      "priorityFee": "0.0005"
    },
    "medium": {
      "gasLimit": "100000",
      "gasPrice": "50000000",
      "maxFee": "0.005",
      "baseFee": "0.004",
      "priorityFee": "0.001"
    },
    "high": {
      "gasLimit": "100000",
      "gasPrice": "80000000",
      "maxFee": "0.008",
      "baseFee": "0.006",
      "priorityFee": "0.002"
    }
  }
}
```

**Note:** All fee amounts are in ETH. Gas limit is estimated based on transaction complexity.

### 7. List Wallets

Retrieve all wallets for your entity, with optional filtering by blockchain or wallet set.

**Endpoint:** `GET /wallets`

**Request:**

```bash
curl -X GET "https://api.circle.com/v1/w3s/wallets?blockchain=BASE&pageSize=10" \
  -H "Authorization: Bearer test_api_key_live_12345abcdef"
```

**Query Parameters:**

- `blockchain`: Filter by blockchain (e.g., "BASE")
- `walletSetId`: Filter by wallet set ID
- `pageSize`: Number of results per page (1-50, default 10)
- `pageBefore`: Cursor for previous page
- `pageAfter`: Cursor for next page

**Response (200 OK):**

```json
{
  "data": {
    "wallets": [
      {
        "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "state": "LIVE",
        "walletSetId": "01234567-89ab-cdef-0123-456789abcdef",
        "custodyType": "DEVELOPER",
        "address": "0x1234567890abcdef1234567890abcdef12345678",
        "blockchain": "BASE",
        "accountType": "SCA",
        "createDate": "2024-01-15T10:35:00Z",
        "updateDate": "2024-01-15T10:35:00Z"
      }
    ]
  },
  "pagination": {
    "nextPageToken": "eyJpZCI6ImIyYzNkNGU1LWY2YTctODkwMS1iY2RlLWYxMjM0NTY3ODkwMSJ9"
  }
}
```

## SDK Usage

### JavaScript/TypeScript SDK

Install the SDK:

```bash
npm install @circle-fin/user-controlled-wallets
```

**Initialize SDK:**

```javascript
const { initiateSdk } = require('@circle-fin/user-controlled-wallets');

const client = initiateSdk({
  apiKey: 'test_api_key_live_12345abcdef',
  entitySecret: 'your_entity_secret_here'
});
```

**Create Wallet:**

```javascript
const response = await client.createWallet({
  blockchains: ['BASE'],
  count: 1,
  walletSetId: '01234567-89ab-cdef-0123-456789abcdef',
  accountType: 'SCA'
});

const walletId = response.data.wallets[0].id;
const address = response.data.wallets[0].address;
```

**Execute Challenge (for user-controlled wallets):**

```javascript
const challenge = await client.executeChallenge({
  userToken: 'user_session_token',
  challengeId: 'challenge_id_from_transaction'
});
```

### Python SDK

Install the SDK:

```bash
pip install circle-programmable-wallets
```

**Initialize Client:**

```python
from circle_programmable_wallets import CircleClient

client = CircleClient(
    api_key='test_api_key_live_12345abcdef',
    entity_secret='your_entity_secret_here'
)
```

**Create Wallet:**

```python
response = client.create_wallet(
    blockchains=['BASE'],
    count=1,
    wallet_set_id='01234567-89ab-cdef-0123-456789abcdef',
    account_type='SCA'
)

wallet_id = response['data']['wallets'][0]['id']
address = response['data']['wallets'][0]['address']
```

**Transfer Tokens:**

```python
transfer_response = client.transfer(
    wallet_id=wallet_id,
    token_id='usdc-base',
    destination_address='0x9876543210fedcba9876543210fedcba98765432',
    amounts=['1000000000'],
    fee_level='MEDIUM'
)

transaction_id = transfer_response['data']['id']
```

## Webhook Notifications

Circle sends webhook notifications for transaction state changes. Configure webhooks in the developer dashboard.

**Webhook Payload Example:**

```json
{
  "subscriptionId": "sub-12345678-90ab-cdef-1234-567890abcdef",
  "notificationId": "notif-abcdef12-3456-7890-abcd-ef1234567890",
  "notificationType": "transactions.transfer",
  "notification": {
    "id": "tx-12345678-90ab-cdef-1234-567890abcdef",
    "state": "COMPLETE",
    "txHash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
    "walletId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "blockchain": "BASE"
  },
  "timestamp": "2024-01-15T11:16:30Z"
}
```

Verify webhook signatures using the signing secret provided in your dashboard.

## Error Codes

All errors follow a consistent format:

```json
{
  "code": 400,
  "message": "Invalid request parameters",
  "errors": [
    {
      "error": "INVALID_BLOCKCHAIN",
      "message": "Blockchain 'INVALID' is not supported"
    }
  ]
}
```

**Common Error Codes:**

| Code | Status | Description | Resolution |
|------|--------|-------------|------------|
| 400 | Bad Request | Invalid request parameters or malformed JSON | Check request body against schema |
| 401 | Unauthorized | Missing or invalid API key | Verify Authorization header |
| 403 | Forbidden | Insufficient permissions or invalid entity secret | Check entity secret encryption |
| 404 | Not Found | Wallet, transaction, or resource not found | Verify resource ID exists |
| 409 | Conflict | Resource already exists or state conflict | Check resource state before operation |
| 422 | Unprocessable Entity | Request valid but cannot be processed (e.g., insufficient balance) | Check wallet balance and transaction parameters |
| 429 | Too Many Requests | Rate limit exceeded | Implement exponential backoff |
| 500 | Internal Server Error | Unexpected server error | Retry with exponential backoff |
| 503 | Service Unavailable | Temporary service outage | Retry after delay |

## Important Addresses

**USDC on Base:**
- Contract: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- Decimals: 6
- Symbol: USDC

**Native ETH:**
- Token ID: `native-eth-base`
- Decimals: 18
- Symbol: ETH

## Additional Resources

- **Smart Wallet Features**: See [smart-wallet skill](../smart-wallet/overview.md) for gas abstraction, batching, and session keys
- **Cross-Chain Transfers**: See [x402 skill](../x402/overview.md) for ERC-7683 cross-chain intents
- **API Status**: Monitor uptime at https://status.circle.com
- **Developer Dashboard**: https://console.circle.com

## Best Practices

1. **Idempotency**: Use idempotency keys for transfer operations to prevent duplicate transactions
2. **Webhook Verification**: Always verify webhook signatures to prevent spoofing
3. **Error Handling**: Implement exponential backoff for rate limits and temporary errors
4. **Balance Checks**: Verify sufficient balance before initiating transfers
5. **Fee Estimation**: Use estimateFee endpoint to show users expected costs
6. **Transaction Monitoring**: Poll transaction status or use webhooks for state updates
7. **Secret Management**: Store entity secrets securely, never commit to version control
8. **Network Selection**: Always specify "BASE" for Base network operations