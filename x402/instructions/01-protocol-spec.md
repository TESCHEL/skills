# x402 Protocol Specification

This document defines the complete technical specification for the x402 Payment-Required Protocol. The protocol enables payment-gated HTTP resources using ERC-20 tokens on Base (and other EVM chains) with EIP-3009 or Permit2 authorization schemes.

## Protocol Overview

The x402 protocol extends HTTP with two custom headers:

- **PAYMENT-REQUIRED**: Sent by servers (402 status) to request payment
- **PAYMENT-RESPONSE**: Sent by clients with payment proof to access resources

Payments use cryptographic signatures over typed data (EIP-712) to authorize token transfers without requiring upfront on-chain transactions. The facilitator processes these authorizations and routes funds to the resource provider.

## Protocol Flow

The complete x402 interaction involves five steps:

### Step 1: Initial Request (No Payment)

Client makes a standard HTTP request to a payment-gated resource:

```
GET /api/premium-data HTTP/1.1
Host: api.example.com
Accept: application/json
```

### Step 2: Payment Required Response

Server responds with 402 status and PAYMENT-REQUIRED header:

```
HTTP/1.1 402 Payment Required
PAYMENT-REQUIRED: eyJhbW91bnQiOiIxMDAwMDAwIiwiY3VycmVuY3kiOiJVU0RDIiwiYWNjZXB0ZWRNZXRob2RzIjpbImVpcDMwMDkiLCJwZXJtaXQyIl0sImV4cGlyeSI6MTczNTA4MDAwMCwicGF5bWVudElkIjoicGF5XzEyMzQ1Njc4OTAiLCJwYXlUbyI6IjB4MmE4QjNkNDU2RWY3ODkwMTIzNGU1NkFiQ0Q3ODkwRjEyMzQ1Njc4OSIsImZhY2lsaXRhdG9yIjoiaHR0cHM6Ly9mYWNpbGl0YXRvci54NDAyLmNvbSIsImNoYWluSWQiOjg0NTMsImRlc2NyaXB0aW9uIjoiQWNjZXNzIHRvIHByZW1pdW0gQVBJIGVuZHBvaW50In0=
Content-Type: application/json

{"error": "Payment required", "message": "This resource requires payment"}
```

### Step 3: Client Prepares Payment

Client decodes the base64 PAYMENT-REQUIRED header and prepares a PaymentPayload with EIP-712 signature:

```typescript
// Decode payment requirements
const paymentRequired = JSON.parse(atob(paymentRequiredHeader));

// Prepare EIP-712 typed data for USDC on Base
const domain = {
  name: "USD Coin",
  version: "2",
  chainId: 8453,
  verifyingContract: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
};

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

const value = {
  from: "0x1234567890AbCdEf1234567890aBcDeF12345678",
  to: "0x2a8B3d456Ef7890123456AbCD7890F1234567890",
  value: "1000000",
  validAfter: 0,
  validBefore: 1735080000,
  nonce: "0x" + crypto.randomBytes(32).toString("hex")
};

// Sign with wallet
const signature = await signer._signTypedData(domain, types, value);
```

### Step 4: Retry Request with Payment

Client retries the original request with PAYMENT-RESPONSE header:

```
GET /api/premium-data HTTP/1.1
Host: api.example.com
Accept: application/json
PAYMENT-RESPONSE: eyJhbW91bnQiOiIxMDAwMDAwIiwiYXNzZXQiOiIweDgzMzU4OWZDRDZlRGI2RTA4ZjRjN0MzMkQ0ZjcxYjU0YmRBMDI5MTMiLCJwYXlUbyI6IjB4MmE4QjNkNDU2RWY3ODkwMTIzNDU2QWJDRDc4OTBGMTIzNDU2Nzg5IiwibWF4VGltZW91dFNlY29uZHMiOjMwMCwiYXV0aG9yaXphdGlvbiI6eyJmcm9tIjoiMHgxMjM0NTY3ODkwQWJDZEVmMTIzNDU2Nzg5MGFCY0RlRjEyMzQ1Njc4IiwidG8iOiIweDJhOEIzZDQ1NkVmNzg5MDEyMzQ1NkFiQ0Q3ODkwRjEyMzQ1Njc4OSIsInZhbHVlIjoiMTAwMDAwMCIsInZhbGlkQWZ0ZXIiOjAsInZhbGlkQmVmb3JlIjoxNzM1MDgwMDAwLCJub25jZSI6IjB4YWJjZGVmMTIzNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4OTBhYmNkZWYxMjM0NTY3ODkwYWJjZGVmMTIzNDU2NzgifSwic2lnbmF0dXJlIjoiMHhhYmNkZWYxMjM0NTY3ODkwYWJjZGVmMTIzNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4OTBhYmNkZWYxMjM0NTY3ODkwYWJjZGVmMTIzNDU2Nzg5MGFiY2RlZjEyMzQ1Njc4OTBhYmNkZWYxMjM0NTY3ODkwYWJjZGVmMTIzNDU2In0=
```

### Step 5: Success Response

Server validates payment, submits to facilitator, and returns requested resource:

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Payment-Settled: {"txHash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890", "status": "confirmed", "settledAmount": "1000000", "settledAt": 1735075000}

{"data": "premium content here"}
```

## PAYMENT-REQUIRED Header Structure

The PAYMENT-REQUIRED header contains a base64-encoded JSON object with the following structure:

```json
{
  "amount": "1000000",
  "currency": "USDC",
  "acceptedMethods": ["eip3009", "permit2"],
  "expiry": 1735080000,
  "paymentId": "pay_1234567890",
  "payTo": "0x2a8B3d456Ef7890123456AbCD7890F1234567890",
  "facilitator": "https://facilitator.x402.com",
  "chainId": 8453,
  "description": "Access to premium API endpoint"
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| amount | string | Yes | Token amount in smallest unit (e.g., 1000000 = 1 USDC with 6 decimals) |
| currency | string | Yes | Human-readable token symbol (USDC, USDT, DAI, etc.) |
| acceptedMethods | string[] | Yes | Payment authorization methods: "eip3009", "permit2" |
| expiry | number | Yes | Unix timestamp when payment requirement expires |
| paymentId | string | Yes | Unique identifier for this payment request (server-generated) |
| payTo | address | Yes | Ethereum address to receive payment (resource provider) |
| facilitator | string | Yes | URL of x402 facilitator service that processes payments |
| chainId | number | Yes | EVM chain ID (8453 for Base mainnet, 84532 for Base Sepolia) |
| description | string | No | Human-readable description of what payment grants access to |

## PaymentPayload Structure

The PAYMENT-RESPONSE header contains a base64-encoded JSON PaymentPayload:

```json
{
  "amount": "1000000",
  "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "payTo": "0x2a8B3d456Ef7890123456AbCD7890F1234567890",
  "maxTimeoutSeconds": 300,
  "authorization": {
    "from": "0x1234567890AbCdEf1234567890aBcDeF12345678",
    "to": "0x2a8B3d456Ef7890123456AbCD7890F1234567890",
    "value": "1000000",
    "validAfter": 0,
    "validBefore": 1735080000,
    "nonce": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890"
  },
  "signature": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef12345678"
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| amount | string | Yes | Must match PAYMENT-REQUIRED amount |
| asset | address | Yes | ERC-20 token contract address |
| payTo | address | Yes | Must match PAYMENT-REQUIRED payTo |
| maxTimeoutSeconds | number | Yes | Maximum seconds to wait for on-chain confirmation |
| authorization | object | Yes | EIP-3009 or Permit2 authorization data |
| signature | string | Yes | EIP-712 signature (0x-prefixed hex, 65 bytes = 130 hex chars) |

## EIP-3009 Authorization Fields

When using the "eip3009" method (supported by USDC on Base), the authorization object contains:

```json
{
  "from": "0x1234567890AbCdEf1234567890aBcDeF12345678",
  "to": "0x2a8B3d456Ef7890123456AbCD7890F1234567890",
  "value": "1000000",
  "validAfter": 0,
  "validBefore": 1735080000,
  "nonce": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890"
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| from | address | Payer's Ethereum address (must be signature signer) |
| to | address | Payee's Ethereum address (facilitator or direct recipient) |
| value | string | Token amount in smallest unit |
| validAfter | number | Unix timestamp after which authorization is valid (0 = immediate) |
| validBefore | number | Unix timestamp before which authorization expires |
| nonce | bytes32 | Unique 32-byte value (0x-prefixed hex) to prevent replay attacks |

## EIP-712 Typed Data for USDC

To sign an EIP-3009 authorization for USDC on Base mainnet:

```typescript
const domain = {
  name: "USD Coin",
  version: "2",
  chainId: 8453,
  verifyingContract: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
};

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

const value = {
  from: userAddress,
  to: facilitatorAddress,
  value: "1000000",
  validAfter: 0,
  validBefore: Math.floor(Date.now() / 1000) + 3600,
  nonce: generateRandomNonce()
};

const signature = await signer._signTypedData(domain, types, value);
```

For Base Sepolia testnet, use chainId 84532 and USDC contract 0x036CbD53842c5426634e7929541eC2318f3dCF7e.

## Permit2 Alternative Method

For tokens that don't support EIP-3009, clients can use Uniswap's Permit2 (0x000000000022D473030F116dDEE9F6B43aC78BA3):

```json
{
  "permitTransferFrom": {
    "permitted": {
      "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "1000000"
    },
    "spender": "0x2a8B3d456Ef7890123456AbCD7890F1234567890",
    "nonce": "123456789",
    "deadline": 1735080000
  },
  "signature": "0xabcdef..."
}
```

Permit2 requires users to first approve the Permit2 contract (one-time transaction). See the smart-wallet skill for automatic approval handling.

## PAYMENT-RESPONSE Header Format

After the facilitator settles payment on-chain, servers MAY include settlement details in the X-Payment-Settled header:

```json
{
  "txHash": "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
  "status": "confirmed",
  "settledAmount": "1000000",
  "settledAt": 1735075000
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| txHash | string | On-chain transaction hash (0x-prefixed hex) |
| status | string | "pending", "confirmed", or "failed" |
| settledAmount | string | Actual amount transferred (may differ due to slippage/fees) |
| settledAt | number | Unix timestamp of on-chain confirmation |

## Chain Configuration

### Base Mainnet (chainId: 8453)

- **USDC Contract**: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
- **Permit2 Contract**: 0x000000000022D473030F116dDEE9F6B43aC78BA3
- **ERC-8004 Registry**: (See erc-8004-registry skill for deployment address)
- **RPC**: https://mainnet.base.org

### Base Sepolia (chainId: 84532)

- **USDC Contract**: 0x036CbD53842c5426634e7929541eC2318f3dCF7e
- **Permit2 Contract**: 0x000000000022D473030F116dDEE9F6B43aC78BA3
- **RPC**: https://sepolia.base.org

## Security Considerations

### Nonce Uniqueness

Every authorization MUST use a unique nonce to prevent replay attacks. The facilitator MUST track used nonces on-chain (via EIP-3009's `authorizationState` or Permit2's nonce tracking).

**Implementation**:
```typescript
function generateRandomNonce(): string {
  return "0x" + crypto.randomBytes(32).toString("hex");
}
```

### Expiry Enforcement

Both the PAYMENT-REQUIRED expiry and the authorization's validBefore MUST be enforced:

1. Client MUST reject expired payment requirements
2. Facilitator MUST reject authorizations with validBefore in the past
3. On-chain contract enforces validBefore automatically

### Facilitator Routing

The facilitator field in PAYMENT-REQUIRED determines where payment authorization is sent. Clients MUST:

1. Verify facilitator URL uses HTTPS
2. Check facilitator's reputation (via ERC-8004 registry)
3. Set reasonable maxTimeoutSeconds (300 seconds recommended)

### Chain Binding

The chainId in both PAYMENT-REQUIRED and EIP-712 domain MUST match. Mismatched chains result in invalid signatures. Cross-reference the smart-wallet skill for multi-chain support.

### Amount Matching

The following amounts MUST be identical:

1. PAYMENT-REQUIRED amount
2. PaymentPayload amount
3. Authorization value
4. Actual on-chain transfer

Mismatches MUST result in payment rejection.

### Signature Verification

Servers or facilitators MUST verify:

1. Signature recovers to authorization.from address
2. EIP-712 domain matches expected contract and chain
3. All authorization fields match PaymentPayload
4. Nonce has not been used previously

### Smart Contract Wallets

For smart contract wallets (EIP-1271), signature verification differs. See the smart-wallet skill for EIP-1271 signature validation using `isValidSignature(bytes32 hash, bytes signature)`.

## Integration Guidelines

### Server Implementation

1. Generate unique paymentId for each request
2. Set reasonable expiry (5-15 minutes typical)
3. Encode PAYMENT-REQUIRED as base64 JSON
4. Return 402 status with header
5. On retry, decode and validate PAYMENT-RESPONSE
6. Submit authorization to facilitator
7. Wait up to maxTimeoutSeconds for confirmation
8. Return 200 with resource or 402 if payment fails

### Client Implementation

1. Detect 402 status code
2. Decode PAYMENT-REQUIRED header
3. Verify expiry, chain, and amount
4. Prompt user for payment approval
5. Generate unique nonce
6. Sign EIP-712 typed data
7. Encode PaymentPayload as base64 JSON
8. Retry request with PAYMENT-RESPONSE header

### Facilitator Implementation

1. Receive authorization from server
2. Validate signature and nonce
3. Check token balance and allowances
4. Call transferWithAuthorization on-chain
5. Wait for transaction confirmation
6. Route payment to payTo address (minus fee)
7. Return settlement details

For complete facilitator implementation, see the smart-wallet and erc-8004-registry skills.

## Error Handling

Servers SHOULD return specific error codes:

- **402**: Payment required or payment failed
- **400**: Invalid PaymentPayload structure
- **408**: Payment timeout (exceeded maxTimeoutSeconds)
- **409**: Nonce already used (replay attack)
- **422**: Amount mismatch or signature invalid

Clients SHOULD handle these gracefully and prompt user appropriately.

## Future Extensions

The x402 protocol is designed for extensibility:

- **Multi-token payments**: acceptedMethods can specify multiple assets
- **Subscription models**: Recurring authorizations with validAfter scheduling
- **Streaming payments**: Continuous micro-payments for real-time data
- **Cross-chain**: Layer 2 rollups and sidechains with bridge support

See the erc-8004-registry skill for protocol versioning and upgrade paths.

---

**Protocol Version**: 1.0.0  
**Last Updated**: 2024-12-29  
**Reference Implementation**: https://github.com/cobuild-lab/x402