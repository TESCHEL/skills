# Basename Registration

## Overview

This lesson covers the complete process of registering .base names through the Basenames Registrar Controller. You will learn how to check name availability, calculate registration costs, submit registration transactions, and verify successful registration. Basenames follow the ENS (Ethereum Name Service) standard adapted for Base L2, providing human-readable names that resolve to Ethereum addresses.

## Registrar Controller Architecture

The Basenames Registrar Controller at 0x4cCb0720c37C1658477b9C7A3aEBb87f1a067292 manages all name registrations and renewals. It interfaces with the underlying registry at 0xb94704422c2a1e396835a571837aa5ae53285a95.

**Key Functions:**
- `available(string name)`: Check if a name is available for registration
- `rentPrice(string name, uint256 duration)`: Calculate cost for registration period
- `register(string name, address owner, uint256 duration, bytes32 secret, address resolver, bytes[] data, bool reverseRecord, uint16 ownerControlledFuses)`: Register a new name
- `renew(string name, uint256 duration)`: Extend registration period

## Setting Up the Client

```typescript
import { createPublicClient, createWalletClient, http, parseEther } from 'viem'
import { base } from 'viem/chains'
import { privateKeyToAccount } from 'viem/accounts'

const publicClient = createPublicClient({
  chain: base,
  transport: http('https://mainnet.base.org')
})

const account = privateKeyToAccount('0xYOUR_PRIVATE_KEY')

const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http('https://mainnet.base.org')
})

const REGISTRAR_CONTROLLER = '0x4cCb0720c37C1658477b9C7A3aEBb87f1a067292'
const REGISTRY = '0xb94704422c2a1e396835a571837aa5ae53285a95'
```

## Checking Name Availability

Before attempting registration, verify the name is available and properly formatted.

```typescript
const registrarControllerAbi = [
  {
    inputs: [{ name: 'name', type: 'string' }],
    name: 'available',
    outputs: [{ name: '', type: 'bool' }],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [
      { name: 'name', type: 'string' },
      { name: 'duration', type: 'uint256' }
    ],
    name: 'rentPrice',
    outputs: [
      { name: 'base', type: 'uint256' },
      { name: 'premium', type: 'uint256' }
    ],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [
      { name: 'name', type: 'string' },
      { name: 'owner', type: 'address' },
      { name: 'duration', type: 'uint256' },
      { name: 'secret', type: 'bytes32' },
      { name: 'resolver', type: 'address' },
      { name: 'data', type: 'bytes[]' },
      { name: 'reverseRecord', type: 'bool' },
      { name: 'ownerControlledFuses', type: 'uint16' }
    ],
    name: 'register',
    outputs: [],
    stateMutability: 'payable',
    type: 'function'
  }
] as const

async function checkAvailability(name: string): Promise<boolean> {
  // Remove .base suffix if present
  const cleanName = name.replace('.base', '')
  
  // Validate name format
  if (!/^[a-z0-9-]{3,63}$/.test(cleanName)) {
    throw new Error('Name must be 3-63 characters: lowercase letters, numbers, hyphens only')
  }
  
  if (cleanName.startsWith('-') || cleanName.endsWith('-')) {
    throw new Error('Name cannot start or end with hyphen')
  }
  
  const available = await publicClient.readContract({
    address: REGISTRAR_CONTROLLER,
    abi: registrarControllerAbi,
    functionName: 'available',
    args: [cleanName]
  })
  
  return available
}

// Example usage
const isAvailable = await checkAvailability('alice')
console.log('alice.base available:', isAvailable)
```

## Calculating Registration Cost

Pricing varies by name length and registration duration.

```typescript
async function getRegistrationPrice(
  name: string,
  durationYears: number
): Promise<{ base: bigint; premium: bigint; total: bigint }> {
  const cleanName = name.replace('.base', '')
  const durationSeconds = BigInt(durationYears * 365 * 24 * 60 * 60)
  
  const [base, premium] = await publicClient.readContract({
    address: REGISTRAR_CONTROLLER,
    abi: registrarControllerAbi,
    functionName: 'rentPrice',
    args: [cleanName, durationSeconds]
  })
  
  return {
    base,
    premium,
    total: base + premium
  }
}

// Example: Get 1-year registration cost
const price = await getRegistrationPrice('alice', 1)
console.log('Base price:', parseEther(price.base.toString()))
console.log('Premium:', parseEther(price.premium.toString()))
console.log('Total cost:', parseEther(price.total.toString()), 'ETH')
```

## Registering a Name

The registration process requires a commitment-reveal scheme for security. For simplicity, we show direct registration (suitable for most use cases).

```typescript
import { keccak256, encodePacked, toBytes } from 'viem'

interface RegistrationParams {
  name: string
  owner: string
  durationYears: number
  resolver?: string
  reverseRecord?: boolean
}

async function registerBasename(params: RegistrationParams): Promise<string> {
  const {
    name,
    owner,
    durationYears,
    resolver = '0x0000000000000000000000000000000000000000', // Default resolver
    reverseRecord = true
  } = params
  
  const cleanName = name.replace('.base', '')
  
  // Check availability
  const available = await checkAvailability(cleanName)
  if (!available) {
    throw new Error(`Name ${cleanName}.base is not available`)
  }
  
  // Get price
  const price = await getRegistrationPrice(cleanName, durationYears)
  const durationSeconds = BigInt(durationYears * 365 * 24 * 60 * 60)
  
  // Generate secret for commitment
  const secret = keccak256(toBytes(`secret-${Date.now()}-${cleanName}`))
  
  // Register
  const hash = await walletClient.writeContract({
    address: REGISTRAR_CONTROLLER,
    abi: registrarControllerAbi,
    functionName: 'register',
    args: [
      cleanName,
      owner as `0x${string}`,
      durationSeconds,
      secret,
      resolver as `0x${string}`,
      [], // No additional data
      reverseRecord,
      0 // No fuses
    ],
    value: price.total
  })
  
  // Wait for confirmation
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  
  if (receipt.status === 'success') {
    return `Successfully registered ${cleanName}.base`
  } else {
    throw new Error('Registration transaction failed')
  }
}

// Example: Register alice.base for 1 year
const result = await registerBasename({
  name: 'alice',
  owner: account.address,
  durationYears: 1,
  reverseRecord: true
})

console.log(result)
```

## Verifying Registration

After registration, verify the name resolves correctly.

```typescript
const registryAbi = [
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'owner',
    outputs: [{ name: '', type: 'address' }],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'resolver',
    outputs: [{ name: '', type: 'address' }],
    stateMutability: 'view',
    type: 'function'
  }
] as const

function namehash(name: string): `0x${string}` {
  const labels = name.split('.')
  let node = '0x0000000000000000000000000000000000000000000000000000000000000000' as `0x${string}`
  
  for (let i = labels.length - 1; i >= 0; i--) {
    const labelHash = keccak256(toBytes(labels[i]))
    node = keccak256(encodePacked(['bytes32', 'bytes32'], [node, labelHash]))
  }
  
  return node
}

async function verifyRegistration(name: string): Promise<{
  owner: string
  resolver: string
  node: string
}> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  const node = namehash(fullName)
  
  const owner = await publicClient.readContract({
    address: REGISTRY,
    abi: registryAbi,
    functionName: 'owner',
    args: [node]
  })
  
  const resolver = await publicClient.readContract({
    address: REGISTRY,
    abi: registryAbi,
    functionName: 'resolver',
    args: [node]
  })
  
  return {
    owner,
    resolver,
    node
  }
}

// Example: Verify alice.base registration
const info = await verifyRegistration('alice.base')
console.log('Owner:', info.owner)
console.log('Resolver:', info.resolver)
console.log('Node:', info.node)
```

## Renewing Registrations

Extend the registration period before expiration.

```typescript
const renewAbi = [
  {
    inputs: [
      { name: 'name', type: 'string' },
      { name: 'duration', type: 'uint256' }
    ],
    name: 'renew',
    outputs: [],
    stateMutability: 'payable',
    type: 'function'
  }
] as const

async function renewBasename(name: string, durationYears: number): Promise<string> {
  const cleanName = name.replace('.base', '')
  const durationSeconds = BigInt(durationYears * 365 * 24 * 60 * 60)
  
  // Get renewal price
  const price = await getRegistrationPrice(cleanName, durationYears)
  
  // Submit renewal
  const hash = await walletClient.writeContract({
    address: REGISTRAR_CONTROLLER,
    abi: renewAbi,
    functionName: 'renew',
    args: [cleanName, durationSeconds],
    value: price.total
  })
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  
  if (receipt.status === 'success') {
    return `Successfully renewed ${cleanName}.base for ${durationYears} year(s)`
  } else {
    throw new Error('Renewal transaction failed')
  }
}

// Example: Renew alice.base for 2 years
const renewResult = await renewBasename('alice', 2)
console.log(renewResult)
```

## Common Patterns

**Bulk Availability Check:**
```typescript
async function checkMultipleNames(names: string[]): Promise<Map<string, boolean>> {
  const results = new Map<string, boolean>()
  
  for (const name of names) {
    try {
      const available = await checkAvailability(name)
      results.set(name, available)
    } catch (error) {
      results.set(name, false)
    }
  }
  
  return results
}

const candidates = ['alice', 'bob', 'charlie']
const availability = await checkMultipleNames(candidates)
availability.forEach((available, name) => {
  console.log(`${name}.base: ${available ? 'Available' : 'Taken'}`)
})
```

**Price Comparison by Length:**
```typescript
async function comparePricesByLength(baseName: string): Promise<void> {
  const lengths = [3, 4, 5, 6, 10]
  
  for (const len of lengths) {
    const testName = baseName.substring(0, len).padEnd(len, 'x')
    const price = await getRegistrationPrice(testName, 1)
    const ethPrice = Number(price.total) / 1e18
    console.log(`${len} chars: ${ethPrice.toFixed(4)} ETH/year`)
  }
}

await comparePricesByLength('test')
```

## Gotchas

**Name Normalization**: Always normalize names to lowercase and remove the .base suffix before passing to contract functions. The registrar expects bare labels.

**Gas Estimation**: Registration transactions can be expensive. Always estimate gas before submitting:
```typescript
const gasEstimate = await publicClient.estimateContractGas({
  address: REGISTRAR_CONTROLLER,
  abi: registrarControllerAbi,
  functionName: 'register',
  args: [/* ... */],
  value: price.total,
  account
})
console.log('Estimated gas:', gasEstimate)
```

**Premium Pricing**: Recently expired names may have premium pricing that decays over time. The `premium` field in `rentPrice` reflects this temporary surcharge.

**Transaction Timing**: During high network activity, registration transactions may take longer to confirm. Always wait for receipt confirmation before assuming success.

**Resolver Configuration**: If you don't specify a resolver during registration, the name won't resolve to an address until you set one. Always use a valid resolver or the default public resolver.