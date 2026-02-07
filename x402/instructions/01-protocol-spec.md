# x402 Protocol Specification

## Overview
x402 is an open standard for internet-native payments built on top of the HTTP 402 Payment Required status code. It allows a client to pay for a resource (API, content, compute) in a single round-trip without prior account setup.

## The Handshake Flow

### 1. Initial Request
The Client requests a protected resource.

GET /api/premium-data HTTP/1.1
Host: api.example.com

### 2. The Challenge (402)
The Server denies access and demands payment.

HTTP/1.1 402 Payment Required
PAYMENT-REQUIRED: <Base64-encoded PaymentRequirements JSON>

Decoded PaymentRequirements Schema:

{
  "accepts": [
    {
      "scheme": "exact",
      "network": "base",
      "maxAmountRequired": "100000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xServerWalletAddress...",
      "resource": "/api/premium-data"
    }
  ]
}

### 3. The Payment (Retry)
The Client signs the payment payload and retries the exact same request with the signature.

GET /api/premium-data HTTP/1.1
Host: api.example.com
PAYMENT-SIGNATURE: <Base64-encoded SignedPaymentPayload>

### 4. Verification & Response
The Server (or Facilitator) verifies the signature, broadcasts the transaction to the blockchain, and serves the content.

HTTP/1.1 200 OK
PAYMENT-RESPONSE: <Base64-encoded TransactionReceipt>
Content-Type: application/json

{ "data": "Here is your premium content" }