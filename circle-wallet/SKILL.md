# Circle Programmable Wallets on Base

## Overview

Circle Programmable Wallets provide enterprise-grade wallet infrastructure for creating and managing wallets on Base. The service enables programmatic wallet creation, transaction signing, and asset management through Circle's API and SDKs, with built-in support for smart contract accounts (ERC-4337) and gasless transactions.

Base Network: Ethereum Layer 2 (Chain ID: 8453)
API Base: https://api.circle.com/v1/w3s
SDKs: @circle-fin/user-controlled-wallets (JavaScript), circle-programmable-wallets (Python)

## Wallet Types

Circle offers two primary wallet architectures:

### User-Controlled Wallets

User-Controlled Wallets give end-users full custody of their private keys. The Circle SDK runs client-side (browser/mobile) and handles key generation, secure storage, and transaction signing locally. Circle's infrastructure never accesses user private keys.

Architecture:
- Client-side key generation via SDK
- Keys encrypted and stored locally (device secure enclave, browser storage)
- User signs transactions on-device
- Circle provides wallet creation and recovery flows

Use cases: Consumer wallets, DeFi apps, gaming wallets where users demand self-custody

### Developer-Controlled Wallets

Developer-Controlled Wallets are managed entirely by Circle's infrastructure using Multi-Party Computation (MPC) and Hardware Security Modules (HSM). Private keys are never exposed -- Circle handles signing via secure API calls.

Architecture:
- Server-side wallet creation via API
- Circle manages keys in distributed MPC network
- Transactions signed via POST /developer/transactions endpoints
- No client-side key management required

Use cases: Custodial wallets, automated treasury operations, server-side payment processing

## Smart Contract Accounts (ERC-4337)

Circle wallets support ERC-4337 Account Abstraction, enabling advanced features like gasless transactions, batch operations, and permission management.

### EntryPoint Contract

All ERC-4337 operations route through the canonical EntryPoint:

Address: 0x0576a174D229E3cFA37253523E645A78A0C91B57
Chain: Base (8453)

The EntryPoint contract validates UserOperations, handles gas payment via Paymasters, and executes account logic.

### Smart Account Features

**Sub Accounts**: App-specific wallets derived from a master account. Each sub-account has independent spending limits and can be frozen or closed without affecting the parent wallet.

**Session Keys**: Temporary permission keys with restricted capabilities (spending limits, token allowlists, time bounds). Useful for gaming sessions or delegated operations.

**Batch Operations**: Execute multiple transactions atomically via executeBatch(). Example: approve token and swap in single UserOperation.

## Gasless Transactions

Circle wallets integrate with Coinbase Paymaster to sponsor gas fees, enabling zero-cost transactions for end-users.

### Coinbase Paymaster Integration

Paymaster URL: https://developer.coinbase.com/paymaster/v1/base
Gas Credits: Up to $15,000 in sponsored gas per developer account

How it works:
1. User creates UserOperation (target, calldata, nonce)
2. SDK requests paymaster signature from Coinbase endpoint
3. Paymaster verifies operation and signs sponsorship
4. UserOperation submitted to EntryPoint with paymaster data
5. Paymaster pays gas; user transaction executes without ETH

Eligibility: USDC transfers, approved smart contract calls, batch operations under $50 value

### Implementing Gasless Transfers

JavaScript example with User-Controlled Wallet:

```javascript
import { initUserControlledWalletSdk } from '@circle-fin/user-controlled-wallets';

const sdk = initUserControlledWalletSdk({ apiKey: 'YOUR_API_KEY' });

// Create gasless USDC transfer
const transfer = await sdk.createTransaction({
  walletId: 'wallet-id-uuid',
  tokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
  destinationAddress: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb',
  amount: '10000000', // 10 USDC (6 decimals)
  usePaymaster: true,
  paymasterUrl: 'https://developer.coinbase.com/paymaster/v1/base'
});

// User signs locally
const signature = await sdk.signTransaction(transfer.userOpHash);

// Submit to EntryPoint
const txHash = await sdk.submitTransaction(transfer.id, signature);
```

Python example with Developer-Controlled Wallet:

```python
from circle_programmable_wallets import DeveloperWallets

client = DeveloperWallets(api_key='YOUR_API_KEY')

# Gasless transfer (Circle signs server-side)
response = client.create_transfer(
    wallet_id='wallet-id-uuid',
    token_address='0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
    destination_address='0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb',
    amount='10000000',
    use_paymaster=True
)

tx_hash = response['data']['transactionHash']
```

## API Reference

### Create Developer Wallet

```
POST https://api.circle.com/v1/w3s/developer/wallets
Authorization: Bearer YOUR_API_KEY

{
  "accountType": "SCA",
  "blockchains": ["BASE"],
  "metadata": [{
    "key": "user_id",
    "value": "user-12345"
  }]
}
```

Response:
```json
{
  "data": {
    "walletId": "01234567-89ab-cdef-0123-456789abcdef",
    "address": "0x8f4e77806FADF6E5F5E9E9F5E5E5E5E5E5E5E5E5",
    "blockchain": "BASE",
    "accountType": "SCA",
    "state": "LIVE"
  }
}
```

### Get Wallet Balances

```
GET https://api.circle.com/v1/w3s/wallets/{walletId}/balances?blockchain=BASE
Authorization: Bearer YOUR_API_KEY
```

Response:
```json
{
  "data": {
    "balances": [
      {
        "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "amount": "150000000",
        "decimals": 6,
        "symbol": "USDC"
      }
    ]
  }
}
```

### Execute Transfer

```
POST https://api.circle.com/v1/w3s/developer/transactions/transfer
Authorization: Bearer YOUR_API_KEY

{
  "walletId": "01234567-89ab-cdef-0123-456789abcdef",
  "tokenAddress": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "destinationAddress": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "amounts": ["10000000"],
  "fee": {
    "type": "paymaster",
    "config": {
      "paymasterUrl": "https://developer.coinbase.com/paymaster/v1/base"
    }
  }
}
```

### Execute Batch Operations

```
POST https://api.circle.com/v1/w3s/developer/transactions/contractExecution
Authorization: Bearer YOUR_API_KEY

{
  "walletId": "01234567-89ab-cdef-0123-456789abcdef",
  "abiFunctionSignature": "executeBatch(address[],bytes[])",
  "abiParameters": [
    ["0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "0xOtherContract"],
    ["0xa9059cbb...", "0x1234abcd..."]
  ],
  "contractAddress": "0xYourSmartAccount",
  "fee": {
    "type": "paymaster"
  }
}
```

## Key Addresses

- USDC on Base: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
- EntryPoint (ERC-4337): 0x0576a174D229E3cFA37253523E645A78A0C91B57
- Base Chain ID: 8453

## Related Skills

- **smart-wallet**: Generic smart contract wallet patterns and ERC-4337 account abstraction
- **x402**: Cross-chain wallet infrastructure and multi-chain asset management

## File Structure

```
instructions/
  01-wallet-types.md         # User-Controlled vs Developer-Controlled architecture
  02-smart-accounts.md       # ERC-4337 integration, Sub Accounts, Session Keys
  03-gasless.md             # Paymaster integration, sponsored transactions
  04-api-reference.md       # Complete API endpoints and SDK methods
```

## Best Practices

1. **User-Controlled for Consumer Apps**: Maximize decentralization and user trust
2. **Developer-Controlled for Backend Services**: Simplify key management for automated operations
3. **Always Use Paymaster for Onboarding**: Remove ETH acquisition friction
4. **Implement Sub Accounts for Multi-App Wallets**: Isolate permissions per use case
5. **Batch Operations for Complex Flows**: Reduce UserOperation overhead (approve + swap -> single batch)
6. **Monitor Paymaster Credits**: Track monthly $15k limit via Coinbase developer dashboard
7. **Test on Base Sepolia First**: Validate flows before mainnet deployment (testnet faucets available)

## Security Considerations

- User-Controlled: Keys never leave client device -- ensure secure storage (keychain, secure enclave)
- Developer-Controlled: API keys grant full wallet control -- use environment variables, rotate regularly
- Smart Accounts: Review EntryPoint contract permissions -- validateUserOp logic is immutable
- Paymaster: Verify sponsorship policies -- malicious contracts may drain paymaster deposits
- Session Keys: Set strict time bounds and spending limits -- revoke immediately after use

## Common Integration Patterns

**Pattern 1: Onboarding Flow**
1. Create User-Controlled Wallet (client-side)
2. Fund via fiat on-ramp (Circle, MoonPay)
3. First transaction uses Paymaster (zero ETH required)
4. User experiences blockchain without crypto knowledge

**Pattern 2: Gaming Economy**
1. Issue Sub Account per game session
2. Grant Session Key with 100 USDC spending limit
3. Game server executes in-app purchases via Session Key
4. Parent wallet retains full control, can revoke anytime

**Pattern 3: Treasury Automation**
1. Deploy Developer-Controlled Wallet for company treasury
2. Schedule recurring USDC transfers via cron + API
3. Use executeBatch for multi-recipient payroll
4. Paymaster sponsors gas -> treasury holds only USDC