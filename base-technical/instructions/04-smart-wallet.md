# Smart Wallet & Account Abstraction on Base

## Overview
Base's Smart Wallet implements ERC-4337 account abstraction. Transactions can be sponsored, batched, and executed without direct gas payments. Coinbase provides Paymaster and Bundler services.

## Key Addresses
- **ERC-4337 Entry Point:** `0x0576a174D229E3cFA37253523E645A78A0C91B57`
- **Paymaster URL:** `https://developer.coinbase.com/paymaster/v1/base`

## Components
### Smart Account Contract
- Contract wallet controlled by owners (EOA or other smart accounts)
- Validates signatures using EIP-1271
- Supports session keys for delegated permissions

### Paymaster
- Sponsors gas for UserOperations
- Up to $15k in gas credits for early adopters
- Obtained via Coinbase Developer Platform API key

### Bundler
- Collects UserOperations, submits to entry point contract
- Run by Coinbase -- developers only call `sendUserOperation`

### Sub Accounts
- App-specific smart wallets under a parent account
- Separate addresses and owners
- Can enforce spending limits and session durations
- Create: `client.createSubAccount({ name: 'BAiSED-Agent' })`

### Session Keys
- Grant limited permissions: spend limit, whitelisted contracts, expiration
- Agent signs operations without exposing main owner key
- Create: `client.createSessionKey({ subAccountAddress, spender: contract, limit: 10*1e6, expiration: Date.now()+3600*1000 })`
- Store session key in agent's secure storage

## SDK Setup
```javascript
import { SmartAccountClient } from 'base-aa-kit';

const client = await SmartAccountClient.init({
  projectId: '<project-id>',
  apiKey: '<api-key>',
  chainId: 8453
});

// Create sub account
const sub = await client.createSubAccount({ name: 'BAiSED-Agent' });

// Add owner
await client.addOwner(sub.address, ownerAddress);

// Send sponsored transaction
await client.paymaster.sendUserOperation({
  subAccountAddress: sub.address,
  target: contractAddress,
  data: calldata,
  value: 0
});
```

## Batch Operations
```javascript
const calls = [
  { target: basenameController, data: registerCalldata, value: registrationFee },
  { target: identityRegistry, data: newAgentCalldata, value: registrationFee },
  { target: resolver, data: setTextCalldata, value: 0 }
];
await client.executeBatch(subAccountAddress, calls);
```
Packages multiple operations into one UserOperation -- reduces fees, ensures atomic execution.

## BAiSED Implementation Plan
1. Create sub account for autonomous operations
2. Keep parent account key offline
3. Use session keys for x402 payments (spending limit + expiration)
4. Integrate paymaster for gasless transactions
5. Use batch calls for multi-step operations (Basename + ERC-8004 + x402 setup)
6. Watch for EIP-7702 (native account abstraction) updates