# OnchainKit Setup Guide

## Installation

Install the required packages:

```bash
npm install @coinbase/onchainkit viem wagmi @tanstack/react-query
```

Or with yarn:

```bash
yarn add @coinbase/onchainkit viem wagmi @tanstack/react-query
```

## Getting Your API Key

1. Visit the [Coinbase Developer Platform](https://portal.cdp.coinbase.com/)
2. Sign in or create an account
3. Navigate to "API Keys" in the dashboard
4. Click "Create API Key"
5. Select the appropriate permissions for your use case
6. Copy your **Client API Key** (this is the public key safe for frontend use)

> **Important**: Never expose your API Secret in frontend code. The Client API Key is designed for browser use.

## Environment Variables

Create a `.env.local` file in your project root:

```env
NEXT_PUBLIC_ONCHAINKIT_API_KEY=your_client_api_key_here
NEXT_PUBLIC_WALLET_CONNECT_PROJECT_ID=your_walletconnect_id_here
```

For WalletConnect (optional but recommended):
1. Visit [WalletConnect Cloud](https://cloud.walletconnect.com/)
2. Create a project and copy the Project ID

## Provider Configuration

Wrap your application with the required providers:

```tsx
// app/providers.tsx
'use client';

import { OnchainKitProvider } from '@coinbase/onchainkit';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { WagmiProvider, createConfig, http } from 'wagmi';
import { base, baseSepolia } from 'wagmi/chains';
import { coinbaseWallet } from 'wagmi/connectors';

const queryClient = new QueryClient();

const wagmiConfig = createConfig({
  chains: [base, baseSepolia],
  connectors: [
    coinbaseWallet({
      appName: 'Your App Name',
      preference: 'smartWalletOnly', // or 'eoaOnly' or 'all'
    }),
  ],
  transports: {
    [base.id]: http(),
    [baseSepolia.id]: http(),
  },
});

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={wagmiConfig}>
      <QueryClientProvider client={queryClient}>
        <OnchainKitProvider
          apiKey={process.env.NEXT_PUBLIC_ONCHAINKIT_API_KEY}
          chain={base} // or baseSepolia for testnet
          config={{
            appearance: {
              mode: 'auto', // 'light' | 'dark' | 'auto'
              theme: 'default', // or custom theme
            },
          }}
        >
          {children}
        </OnchainKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

## Required CSS Import

Add the OnchainKit styles to your root layout:

```tsx
// app/layout.tsx
import '@coinbase/onchainkit/styles.css';
import './globals.css'; // Your styles after OnchainKit

import { Providers } from './providers';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

> **Note**: Import OnchainKit styles BEFORE your custom styles to allow proper CSS cascade overrides.

## Verifying Your Setup

Test your configuration with a simple wallet connection:

```tsx
// app/page.tsx
import { Wallet, ConnectWallet } from '@coinbase/onchainkit/wallet';

export default function Home() {
  return (
    <main>
      <Wallet>
        <ConnectWallet />
      </Wallet>
    </main>
  );
}
```

If you see the Connect Wallet button and can connect, your setup is complete!

## Next Steps

- [Identity Components](./02-identity.md) - Display user profiles
- [Wallet Integration](./03-wallet.md) - Advanced wallet features
- [Payments](./04-payments.md) - Accept crypto payments
