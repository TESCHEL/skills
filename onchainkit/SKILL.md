---
name: onchainkit
version: 1.0.0
description: Build onchain apps with OnchainKit - a React component library for Base
author: BAiSEDagent
tags: [base, onchain, react, web3, coinbase]
---

# OnchainKit Skill

Build onchain applications using OnchainKit, a React component library by Coinbase for the Base network.

## Overview

OnchainKit provides ready-to-use React components for common onchain operations:
- **Wallet Connection** - Connect wallets with WalletDefault component
- **Identity** - Display ENS names, avatars, and addresses
- **Transactions** - Build and send transactions with TransactionDefault
- **Swap** - Token swapping interface with SwapDefault
- **Checkout** - USDC payment flows with CheckoutDefault
- **Fund** - Onramp fiat to crypto with FundDefault
- **NFT** - Mint and display NFTs with NFTCard and NFTMintCard
- **Earn** - DeFi yield opportunities with EarnDefault

## Quick Start

```bash
npm install @coinbase/onchainkit
```

```tsx
import { OnchainKitProvider } from '@coinbase/onchainkit';
import { base } from 'wagmi/chains';

function App() {
  return (
    <OnchainKitProvider
      apiKey={process.env.ONCHAINKIT_API_KEY}
      chain={base}
    >
      {children}
    </OnchainKitProvider>
  );
}
```

## Key Components

### Wallet Connection
```tsx
import { WalletDefault } from '@coinbase/onchainkit/wallet';
<WalletDefault />
```

### Identity Display
```tsx
import { Identity, Name, Avatar, Address } from '@coinbase/onchainkit/identity';
<Identity address="0x...">
  <Avatar />
  <Name />
  <Address />
</Identity>
```

### Transactions
```tsx
import { TransactionDefault } from '@coinbase/onchainkit/transaction';
<TransactionDefault
  contracts={[{
    address: '0x...',
    abi: contractAbi,
    functionName: 'mint',
    args: [1]
  }]}
/>
```

### Token Swap
```tsx
import { SwapDefault } from '@coinbase/onchainkit/swap';
<SwapDefault
  from={['ETH']}
  to={['USDC', 'DAI']}
/>
```

## Resources

- [Documentation](https://onchainkit.xyz)
- [GitHub](https://github.com/coinbase/onchainkit)
- [Playground](https://onchainkit.xyz/playground)
- [Templates](https://onchainkit.xyz/templates)
