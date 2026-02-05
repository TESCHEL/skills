# Identity Components

OnchainKit provides powerful identity components for displaying user profiles, resolving Basenames, and showing avatars.

## Basic Identity Display

```tsx
import { Identity, Avatar, Name, Address } from '@coinbase/onchainkit/identity';

function UserProfile({ address }: { address: `0x${string}` }) {
  return (
    <Identity
      address={address}
      schemaId="0xf8b05c79f090979bf4a80270aba232dff11a10d9ca55c4f88de95317970f0de9"
    >
      <Avatar />
      <Name />
      <Address />
    </Identity>
  );
}
```

## Understanding Basenames

Basenames are human-readable names on Base (like ENS for Ethereum). They follow the format `name.base.eth`.

### Resolution Priority

OnchainKit resolves names in this order:
1. **Basenames** (on Base chain) - e.g., `vitalik.base.eth`
2. **ENS** (on Ethereum mainnet) - e.g., `vitalik.eth`
3. **Truncated address** - e.g., `0x1234...5678`

## Avatar Component

Display user avatars with fallback support:

```tsx
import { Avatar } from '@coinbase/onchainkit/identity';

// Basic usage
<Avatar address="0x..." />

// With custom size
<Avatar 
  address="0x..."
  className="w-12 h-12 rounded-full"
/>

// With loading state
<Avatar 
  address="0x..."
  loadingComponent={<Skeleton />}
  defaultComponent={<DefaultAvatar />}
/>
```

### Avatar Sources

Avatars are resolved from:
1. Basename avatar (set on Base)
2. ENS avatar
3. Fallback gradient based on address

## Name Component

Display resolved names:

```tsx
import { Name } from '@coinbase/onchainkit/identity';

// Basic usage
<Name address="0x..." />

// With custom badge for attestations
<Name 
  address="0x..."
  showAttestation
/>

// Handling loading states
<Name 
  address="0x..."
  loadingComponent={<span>Loading...</span>}
/>
```

## Handling ENS vs Basenames

When building cross-chain apps, handle both name systems:

```tsx
import { useIdentity } from '@coinbase/onchainkit/identity';

function CrossChainIdentity({ address }: { address: `0x${string}` }) {
  // OnchainKit handles resolution automatically
  // Shows Basename if on Base, ENS if on Ethereum
  return (
    <Identity address={address}>
      <Avatar />
      <Name />
    </Identity>
  );
}
```

### Manual Resolution

For custom logic, use the hooks directly:

```tsx
import { useName, useAvatar } from '@coinbase/onchainkit/identity';

function CustomProfile({ address }: { address: `0x${string}` }) {
  const { data: name, isLoading: nameLoading } = useName({ address });
  const { data: avatar, isLoading: avatarLoading } = useAvatar({ address });

  if (nameLoading || avatarLoading) {
    return <LoadingSkeleton />;
  }

  return (
    <div className="flex items-center gap-2">
      {avatar && <img src={avatar} alt={name || 'Avatar'} />}
      <span>{name || `${address.slice(0, 6)}...${address.slice(-4)}`}</span>
    </div>
  );
}
```

## Address Component

Display formatted addresses:

```tsx
import { Address } from '@coinbase/onchainkit/identity';

// Shows truncated address: 0x1234...5678
<Address address="0x..." />

// Full address display
<Address address="0x..." isSliced={false} />
```

## Full Example

Complete identity card component:

```tsx
import { 
  Identity, 
  Avatar, 
  Name, 
  Address,
  Badge 
} from '@coinbase/onchainkit/identity';

function IdentityCard({ address }: { address: `0x${string}` }) {
  return (
    <div className="p-4 border rounded-lg shadow-sm">
      <Identity
        address={address}
        className="flex items-center gap-4"
      >
        <Avatar className="w-16 h-16 rounded-full" />
        <div className="flex flex-col">
          <div className="flex items-center gap-2">
            <Name className="font-bold text-lg" />
            <Badge />
          </div>
          <Address className="text-sm text-gray-500" />
        </div>
      </Identity>
    </div>
  );
}
```

## Attestations

Display verified credentials:

```tsx
import { Identity, Badge } from '@coinbase/onchainkit/identity';

// The Badge component shows attestation status
// Requires schemaId prop on Identity
<Identity
  address={address}
  schemaId="0xf8b05c79f090979bf4a80270aba232dff11a10d9ca55c4f88de95317970f0de9"
>
  <Avatar />
  <Name />
  <Badge />
</Identity>
```

## Next Steps

- [Wallet Integration](./03-wallet.md) - Connect wallets
- [Payments](./04-payments.md) - Accept payments
