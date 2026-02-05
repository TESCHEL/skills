# Wallet Integration

OnchainKit provides comprehensive wallet components for connecting users, handling transactions, and supporting both Smart Wallets and EOAs.

## Basic Wallet Setup

```tsx
import { 
  Wallet, 
  ConnectWallet, 
  WalletDropdown,
  WalletDropdownDisconnect 
} from '@coinbase/onchainkit/wallet';
import { Avatar, Name, Address } from '@coinbase/onchainkit/identity';

function WalletComponent() {
  return (
    <Wallet>
      <ConnectWallet>
        <Avatar />
        <Name />
      </ConnectWallet>
      <WalletDropdown>
        <Address />
        <WalletDropdownDisconnect />
      </WalletDropdown>
    </Wallet>
  );
}
```

## Smart Wallet vs EOA

### Understanding the Difference

| Feature | Smart Wallet | EOA (Externally Owned Account) |
|---------|-------------|-------------------------------|
| Key Storage | Passkey/biometric | Browser extension/hardware |
| Gas Sponsorship | Native support | Requires relayer |
| Batch Transactions | Yes | No |
| Session Keys | Yes | No |
| Recovery | Social/passkey | Seed phrase |

### Configuring Wallet Preference

```tsx
import { coinbaseWallet } from 'wagmi/connectors';

const wagmiConfig = createConfig({
  chains: [base],
  connectors: [
    coinbaseWallet({
      appName: 'Your App',
      // Choose wallet type:
      preference: 'smartWalletOnly', // Only Smart Wallet
      // preference: 'eoaOnly',      // Only Coinbase Wallet extension
      // preference: 'all',          // Let user choose
    }),
  ],
  transports: {
    [base.id]: http(),
  },
});
```

## Connect/Disconnect Flows

### Custom Connect Button

```tsx
import { ConnectWallet, ConnectWalletText } from '@coinbase/onchainkit/wallet';

function CustomConnect() {
  return (
    <Wallet>
      <ConnectWallet className="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-lg">
        <ConnectWalletText>Sign In</ConnectWalletText>
      </ConnectWallet>
    </Wallet>
  );
}
```

### Handling Connection State

```tsx
import { useAccount, useDisconnect } from 'wagmi';

function ConnectionManager() {
  const { address, isConnected, isConnecting } = useAccount();
  const { disconnect } = useDisconnect();

  if (isConnecting) {
    return <div>Connecting...</div>;
  }

  if (isConnected) {
    return (
      <div>
        <p>Connected: {address}</p>
        <button onClick={() => disconnect()}>Disconnect</button>
      </div>
    );
  }

  return <ConnectWallet />;
}
```

## Gasless Transactions with Paymaster

Enable sponsored transactions so users don't pay gas:

### 1. Configure Paymaster in Provider

```tsx
<OnchainKitProvider
  apiKey={process.env.NEXT_PUBLIC_ONCHAINKIT_API_KEY}
  chain={base}
  config={{
    paymaster: process.env.NEXT_PUBLIC_PAYMASTER_URL,
    // Or use Coinbase's paymaster:
    // paymaster: 'https://api.developer.coinbase.com/rpc/v1/base/...'
  }}
>
  {children}
</OnchainKitProvider>
```

### 2. Using Transaction Component with Sponsorship

```tsx
import { Transaction, TransactionButton } from '@coinbase/onchainkit/transaction';

function SponsoredTransaction() {
  const contracts = [{
    address: '0x...',
    abi: contractAbi,
    functionName: 'mint',
    args: [1],
  }];

  return (
    <Transaction
      contracts={contracts}
      onSuccess={(response) => console.log('Success!', response)}
    >
      <TransactionButton text="Mint (Free)" />
    </Transaction>
  );
}
```

### 3. Setting Up Your Paymaster

For Coinbase Developer Platform paymaster:

1. Go to [CDP Portal](https://portal.cdp.coinbase.com/)
2. Navigate to "Paymaster" section
3. Create a new paymaster policy
4. Copy the paymaster URL
5. Add to your environment variables

## Session Keys for Agents

Session keys allow automated transactions without user approval for each action - perfect for AI agents.

### Requesting Session Key

```tsx
import { useGrantPermissions } from 'wagmi/experimental';

function AgentSession() {
  const { grantPermissions } = useGrantPermissions();

  const requestSession = async () => {
    const permissions = await grantPermissions({
      permissions: [{
        type: 'contract-call',
        data: {
          address: '0x...', // Contract address
          calls: [{
            function: 'transfer(address,uint256)',
          }],
        },
        policies: [{
          type: 'spending-limit',
          data: {
            token: '0x...', // Token address
            limit: '1000000000000000000', // 1 token in wei
          },
        }],
        required: true,
      }],
      expiry: Math.floor(Date.now() / 1000) + 86400, // 24 hours
    });

    return permissions;
  };

  return (
    <button onClick={requestSession}>
      Grant Agent Permissions
    </button>
  );
}
```

### Using Session Key for Transactions

```tsx
import { useSendCalls } from 'wagmi/experimental';

function AgentTransaction({ sessionKey }: { sessionKey: string }) {
  const { sendCalls } = useSendCalls();

  const executeAgentAction = async () => {
    await sendCalls({
      calls: [{
        to: '0x...',
        data: '0x...',
        value: 0n,
      }],
      capabilities: {
        permissions: {
          context: sessionKey,
        },
      },
    });
  };

  return (
    <button onClick={executeAgentAction}>
      Execute Agent Action
    </button>
  );
}
```

## Complete Wallet Example

```tsx
import {
  Wallet,
  ConnectWallet,
  WalletDropdown,
  WalletDropdownLink,
  WalletDropdownDisconnect,
} from '@coinbase/onchainkit/wallet';
import { Avatar, Name, Address, Identity } from '@coinbase/onchainkit/identity';

function FullWalletComponent() {
  return (
    <Wallet>
      <ConnectWallet className="btn-primary">
        <Avatar className="w-6 h-6" />
        <Name />
      </ConnectWallet>
      <WalletDropdown>
        <Identity className="px-4 pt-3 pb-2">
          <Avatar />
          <Name />
          <Address />
        </Identity>
        <WalletDropdownLink 
          icon="wallet" 
          href="https://wallet.coinbase.com"
        >
          View Wallet
        </WalletDropdownLink>
        <WalletDropdownDisconnect />
      </WalletDropdown>
    </Wallet>
  );
}
```

## Next Steps

- [Payments](./04-payments.md) - Accept crypto payments
- [Troubleshooting](./05-troubleshooting.md) - Common issues
