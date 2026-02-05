# Troubleshooting OnchainKit

Common issues and solutions when working with OnchainKit.

## API Errors

### 401 Unauthorized

**Symptoms:**
- API calls fail with 401 status
- Components don't load data
- "Unauthorized" errors in console

**Solutions:**

1. **Check API Key Configuration**
```tsx
// Ensure API key is set in OnchainKitProvider
<OnchainKitProvider
  apiKey={process.env.NEXT_PUBLIC_ONCHAINKIT_API_KEY}
  // ...
>
```

2. **Verify Environment Variable**
```bash
# .env.local
NEXT_PUBLIC_ONCHAINKIT_API_KEY=your_actual_api_key

# Check it's not undefined
console.log('API Key:', process.env.NEXT_PUBLIC_ONCHAINKIT_API_KEY);
```

3. **Check API Key Permissions**
   - Visit [CDP Portal](https://portal.cdp.coinbase.com/)
   - Verify the key has appropriate permissions
   - Ensure it's a Client API Key (not Server API Key)

4. **Key Not Loading in Browser**
   - Restart dev server after adding env vars
   - Ensure variable starts with `NEXT_PUBLIC_` for Next.js

### 403 Forbidden

**Possible Causes:**
- Rate limiting
- API key disabled
- Wrong API endpoint

**Solutions:**
- Check rate limits on CDP dashboard
- Verify API key status is active
- Ensure you're using Base chain endpoints

## CSS Not Loading

**Symptoms:**
- Components render without styling
- Buttons look like plain HTML
- Layout is broken

**Solutions:**

1. **Import CSS in Correct Order**
```tsx
// app/layout.tsx
import '@coinbase/onchainkit/styles.css'; // FIRST
import './globals.css'; // Your styles AFTER
```

2. **Check Import Path**
```tsx
// Correct
import '@coinbase/onchainkit/styles.css';

// Wrong
import '@coinbase/onchainkit/dist/styles.css';
import 'onchainkit/styles.css';
```

3. **Verify Package Installation**
```bash
# Check OnchainKit is installed
npm list @coinbase/onchainkit

# Reinstall if needed
npm install @coinbase/onchainkit@latest
```

4. **Tailwind Conflicts**
```js
// tailwind.config.js
module.exports = {
  content: [
    // Include OnchainKit in content paths
    './node_modules/@coinbase/onchainkit/**/*.{js,ts,jsx,tsx}',
    // ...your paths
  ],
  // Don't prefix OnchainKit classes
  important: false, // or use selector strategy
};
```

## Chain Mismatch Errors

**Symptoms:**
- "Wrong network" errors
- Transactions fail silently
- Components show wrong data

**Solutions:**

1. **Match Provider and Wagmi Config**
```tsx
// Both must use the same chain
const wagmiConfig = createConfig({
  chains: [base], // Base mainnet
  // ...
});

<OnchainKitProvider
  chain={base} // Must match!
>
```

2. **Handle Network Switching**
```tsx
import { useSwitchChain } from 'wagmi';
import { base } from 'wagmi/chains';

function NetworkCheck() {
  const { chainId } = useAccount();
  const { switchChain } = useSwitchChain();

  if (chainId !== base.id) {
    return (
      <button onClick={() => switchChain({ chainId: base.id })}>
        Switch to Base
      </button>
    );
  }

  return <YourComponent />;
}
```

3. **Testnet vs Mainnet**
```tsx
// Development - use testnet
const chain = process.env.NODE_ENV === 'development' 
  ? baseSepolia 
  : base;

<OnchainKitProvider chain={chain}>
```

## Debugging Tips

### Enable Verbose Logging

```tsx
// Add to your provider setup
<OnchainKitProvider
  config={{
    debug: process.env.NODE_ENV === 'development',
  }}
>
```

### Check Network Requests

1. Open browser DevTools â Network tab
2. Filter by "Fetch/XHR"
3. Look for failed requests to:
   - `api.developer.coinbase.com`
   - `base-mainnet.g.alchemy.com`
   - `cloudflare-eth.com`

### Inspect Component State

```tsx
import { useAccount, useChainId } from 'wagmi';

function DebugInfo() {
  const { address, isConnected, connector } = useAccount();
  const chainId = useChainId();

  console.log('Debug:', {
    address,
    isConnected,
    chainId,
    connector: connector?.name,
  });

  return null;
}
```

### Transaction Debugging

```tsx
<Transaction
  contracts={contracts}
  onStatus={(status) => console.log('Status:', status)}
  onSuccess={(response) => {
    console.log('Success:', response);
    console.log('TX Hash:', response.transactionReceipts?.[0]?.transactionHash);
  }}
  onError={(error) => {
    console.error('Error:', error);
    console.error('Error Code:', error.code);
    console.error('Error Message:', error.message);
  }}
>
```

## Mobile Responsiveness

### Common Issues

1. **Wallet Modal Not Appearing**
```tsx
// Ensure viewport meta tag
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

2. **Touch Events Not Working**
```css
/* Add to your CSS */
button, [role="button"] {
  touch-action: manipulation;
}
```

3. **Modal Positioning**
```css
/* Fix modal positioning on mobile */
[data-onchainkit-modal] {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 9999;
}
```

### Mobile Wallet Deep Links

```tsx
// Ensure mobile wallets can connect
const wagmiConfig = createConfig({
  connectors: [
    coinbaseWallet({
      appName: 'Your App',
      preference: 'all', // Allow both Smart Wallet and extension
      appLogoUrl: 'https://yourapp.com/logo.png', // Shown in wallet
    }),
  ],
});
```

## Wagmi Hook Issues

### "No QueryClient set" Error

```tsx
// Ensure QueryClientProvider wraps everything
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <WagmiProvider config={wagmiConfig}>
        <OnchainKitProvider>
          {children}
        </OnchainKitProvider>
      </WagmiProvider>
    </QueryClientProvider>
  );
}
```

### Hooks Called Outside Provider

**Error:** "useAccount must be used within WagmiProvider"

```tsx
// Wrong - hook outside provider
function App() {
  const { address } = useAccount(); // Error!
  return <Providers><Child /></Providers>;
}

// Correct - hook inside provider tree
function App() {
  return (
    <Providers>
      <Child /> {/* useAccount works here */}
    </Providers>
  );
}
```

### Hydration Errors

```tsx
'use client';

import { useAccount } from 'wagmi';
import { useEffect, useState } from 'react';

function WalletStatus() {
  const { address, isConnected } = useAccount();
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  // Prevent hydration mismatch
  if (!mounted) {
    return <div>Loading...</div>;
  }

  return <div>{isConnected ? address : 'Not connected'}</div>;
}
```

### Stale Data Issues

```tsx
import { useQueryClient } from '@tanstack/react-query';

function RefreshData() {
  const queryClient = useQueryClient();

  const refresh = () => {
    // Invalidate all queries
    queryClient.invalidateQueries();
    
    // Or specific queries
    queryClient.invalidateQueries({ queryKey: ['balance'] });
  };

  return <button onClick={refresh}>Refresh</button>;
}
```

## Still Having Issues?

1. **Check OnchainKit GitHub Issues**
   - [github.com/coinbase/onchainkit/issues](https://github.com/coinbase/onchainkit/issues)

2. **Join the Community**
   - [Discord](https://discord.gg/cdp)
   - [Warpcast](https://warpcast.com/~/channel/onchainkit)

3. **Check Version Compatibility**
```bash
npm list @coinbase/onchainkit viem wagmi @tanstack/react-query
```

Ensure versions are compatible:
- OnchainKit 0.36+ requires Wagmi 2.x
- Wagmi 2.x requires Viem 2.x
- TanStack Query 5.x recommended
