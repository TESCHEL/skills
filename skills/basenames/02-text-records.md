# Text Records Management

## Overview

Text records allow you to associate metadata with your basename, including avatars, social media handles, URLs, and custom key-value pairs. This lesson covers setting, reading, and updating text records through the resolver contract. Text records follow the ENS text record standard, enabling interoperability with ENS-compatible applications.

## Text Record Architecture

Text records are stored in resolver contracts, not the main registry. The resolver acts as a flexible key-value store where keys are standardized strings (e.g., "avatar", "com.twitter") and values are arbitrary text.

**Standard Text Record Keys:**
- `avatar`: URL or NFT identifier for profile picture
- `description`: Brief bio or description
- `url`: Personal or project website
- `email`: Contact email address
- `com.twitter`: Twitter/X handle (without @)
- `com.github`: GitHub username
- `com.discord`: Discord handle
- `com.telegram`: Telegram username
- Custom keys: Any string for application-specific data

## Resolver Contract Setup

First, get the resolver address for a name, then interact with it to manage records.

```typescript
import { createPublicClient, createWalletClient, http, keccak256, encodePacked, toBytes } from 'viem'
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

const REGISTRY = '0xb94704422c2a1e396835a571837aa5ae53285a95'

function namehash(name: string): `0x${string}` {
  const labels = name.split('.')
  let node = '0x0000000000000000000000000000000000000000000000000000000000000000' as `0x${string}`
  
  for (let i = labels.length - 1; i >= 0; i--) {
    const labelHash = keccak256(toBytes(labels[i]))
    node = keccak256(encodePacked(['bytes32', 'bytes32'], [node, labelHash]))
  }
  
  return node
}

const registryAbi = [
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'resolver',
    outputs: [{ name: '', type: 'address' }],
    stateMutability: 'view',
    type: 'function'
  }
] as const

async function getResolver(name: string): Promise<string> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  const node = namehash(fullName)
  
  const resolver = await publicClient.readContract({
    address: REGISTRY,
    abi: registryAbi,
    functionName: 'resolver',
    args: [node]
  })
  
  return resolver
}

// Example: Get resolver for alice.base
const resolverAddress = await getResolver('alice.base')
console.log('Resolver:', resolverAddress)
```

## Reading Text Records

Retrieve existing text records from the resolver.

```typescript
const resolverAbi = [
  {
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'key', type: 'string' }
    ],
    name: 'text',
    outputs: [{ name: '', type: 'string' }],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'key', type: 'string' },
      { name: 'value', type: 'string' }
    ],
    name: 'setText',
    outputs: [],
    stateMutability: 'nonpayable',
    type: 'function'
  },
  {
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'contentTypes', type: 'uint256' }
    ],
    name: 'addr',
    outputs: [{ name: '', type: 'bytes' }],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'addr',
    outputs: [{ name: '', type: 'address' }],
    stateMutability: 'view',
    type: 'function'
  }
] as const

async function getTextRecord(
  name: string,
  key: string
): Promise<string> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  const node = namehash(fullName)
  const resolver = await getResolver(name)
  
  if (resolver === '0x0000000000000000000000000000000000000000') {
    throw new Error('No resolver set for this name')
  }
  
  const value = await publicClient.readContract({
    address: resolver as `0x${string}`,
    abi: resolverAbi,
    functionName: 'text',
    args: [node, key]
  })
  
  return value
}

// Example: Get Twitter handle for alice.base
const twitter = await getTextRecord('alice.base', 'com.twitter')
console.log('Twitter:', twitter)

// Get multiple records
async function getAllTextRecords(name: string): Promise<Map<string, string>> {
  const standardKeys = [
    'avatar',
    'description',
    'url',
    'email',
    'com.twitter',
    'com.github',
    'com.discord',
    'com.telegram'
  ]
  
  const records = new Map<string, string>()
  
  for (const key of standardKeys) {
    try {
      const value = await getTextRecord(name, key)
      if (value) {
        records.set(key, value)
      }
    } catch (error) {
      // Skip keys that error or are not set
    }
  }
  
  return records
}

const allRecords = await getAllTextRecords('alice.base')
allRecords.forEach((value, key) => {
  console.log(`${key}: ${value}`)
})
```

## Setting Text Records

Update or create new text records. You must be the owner of the name to set records.

```typescript
async function setTextRecord(
  name: string,
  key: string,
  value: string
): Promise<string> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  const node = namehash(fullName)
  const resolver = await getResolver(name)
  
  if (resolver === '0x0000000000000000000000000000000000000000') {
    throw new Error('No resolver set for this name')
  }
  
  const hash = await walletClient.writeContract({
    address: resolver as `0x${string}`,
    abi: resolverAbi,
    functionName: 'setText',
    args: [node, key, value]
  })
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  
  if (receipt.status === 'success') {
    return `Successfully set ${key} to "${value}" for ${fullName}`
  } else {
    throw new Error('Set text record transaction failed')
  }
}

// Example: Set Twitter handle
const result = await setTextRecord('alice.base', 'com.twitter', 'alice_crypto')
console.log(result)

// Example: Set avatar (IPFS or HTTP URL)
await setTextRecord('alice.base', 'avatar', 'ipfs://QmXxxx...')

// Example: Set website
await setTextRecord('alice.base', 'url', 'https://alice.example.com')
```

## Batch Setting Records

Set multiple text records in a single transaction using multicall patterns.

```typescript
interface TextRecordUpdate {
  key: string
  value: string
}

async function setMultipleTextRecords(
  name: string,
  records: TextRecordUpdate[]
): Promise<string> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  const node = namehash(fullName)
  const resolver = await getResolver(name)
  
  if (resolver === '0x0000000000000000000000000000000000000000') {
    throw new Error('No resolver set for this name')
  }
  
  // Set records sequentially (in production, use multicall for efficiency)
  const results: string[] = []
  
  for (const record of records) {
    const hash = await walletClient.writeContract({
      address: resolver as `0x${string}`,
      abi: resolverAbi,
      functionName: 'setText',
      args: [node, record.key, record.value]
    })
    
    const receipt = await publicClient.waitForTransactionReceipt({ hash })
    
    if (receipt.status === 'success') {
      results.push(`Set ${record.key}`)
    } else {
      results.push(`Failed to set ${record.key}`)
    }
  }
  
  return `Updated ${results.length} records for ${fullName}: ${results.join(', ')}`
}

// Example: Set social media profile
const updates = [
  { key: 'com.twitter', value: 'alice_crypto' },
  { key: 'com.github', value: 'alice' },
  { key: 'description', value: 'DeFi enthusiast and builder on Base' },
  { key: 'url', value: 'https://alice.example.com' }
]

const batchResult = await setMultipleTextRecords('alice.base', updates)
console.log(batchResult)
```

## Avatar Records (Special Handling)

Avatars support multiple formats: HTTP URLs, IPFS hashes, and NFT identifiers.

```typescript
interface AvatarConfig {
  type: 'url' | 'ipfs' | 'nft'
  value: string
}

function formatAvatar(config: AvatarConfig): string {
  switch (config.type) {
    case 'url':
      return config.value
    case 'ipfs':
      // IPFS format: ipfs://CID
      return config.value.startsWith('ipfs://') 
        ? config.value 
        : `ipfs://${config.value}`
    case 'nft':
      // NFT format: eip155:chainId/contractAddress/tokenId
      // Example: eip155:8453/0x1234.../1
      return config.value
    default:
      return config.value
  }
}

async function setAvatar(name: string, config: AvatarConfig): Promise<string> {
  const avatarValue = formatAvatar(config)
  return setTextRecord(name, 'avatar', avatarValue)
}

// Example: Set HTTP avatar
await setAvatar('alice.base', {
  type: 'url',
  value: 'https://example.com/avatar.png'
})

// Example: Set IPFS avatar
await setAvatar('alice.base', {
  type: 'ipfs',
  value: 'QmXxxx...'
})

// Example: Set NFT avatar
await setAvatar('alice.base', {
  type: 'nft',
  value: 'eip155:8453/0x1234567890123456789012345678901234567890/1'
})
```

## Custom Application Records

Create custom text records for application-specific data.

```typescript
async function setCustomRecord(
  name: string,
  appName: string,
  key: string,
  value: string
): Promise<string> {
  // Use reverse domain notation for custom keys
  const customKey = `${appName}.${key}`
  return setTextRecord(name, customKey, value)
}

// Example: Store DeFi preferences
await setCustomRecord('alice.base', 'app.defi', 'preferred_dex', 'aerodrome')
await setCustomRecord('alice.base', 'app.defi', 'slippage_tolerance', '0.5')

// Example: Store gaming data
await setCustomRecord('alice.base', 'game.racing', 'high_score', '9999')
await setCustomRecord('alice.base', 'game.racing', 'favorite_track', 'base_boulevard')

// Read custom record
const preferredDex = await getTextRecord('alice.base', 'app.defi.preferred_dex')
console.log('Preferred DEX:', preferredDex)
```

## Clearing Text Records

Remove a text record by setting it to an empty string.

```typescript
async function clearTextRecord(name: string, key: string): Promise<string> {
  return setTextRecord(name, key, '')
}

// Example: Clear Twitter handle
await clearTextRecord('alice.base', 'com.twitter')

// Clear multiple records
async function clearMultipleRecords(name: string, keys: string[]): Promise<void> {
  for (const key of keys) {
    await clearTextRecord(name, key)
    console.log(`Cleared ${key}`)
  }
}

await clearMultipleRecords('alice.base', ['com.discord', 'com.telegram'])
```

## Validating Text Records

Validate text records before setting to ensure data quality.

```typescript
interface ValidationResult {
  valid: boolean
  error?: string
}

function validateTextRecord(key: string, value: string): ValidationResult {
  // URL validation
  if (key === 'url' || key === 'avatar') {
    try {
      if (value.startsWith('ipfs://')) {
        return { valid: true }
      }
      new URL(value)
      return { valid: true }
    } catch {
      return { valid: false, error: 'Invalid URL format' }
    }
  }
  
  // Twitter handle validation (no @ symbol)
  if (key === 'com.twitter') {
    if (value.startsWith('@')) {
      return { valid: false, error: 'Remove @ symbol from Twitter handle' }
    }
    if (!/^[A-Za-z0-9_]{1,15}$/.test(value)) {
      return { valid: false, error: 'Invalid Twitter handle format' }
    }
  }
  
  // Email validation
  if (key === 'email') {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    if (!emailRegex.test(value)) {
      return { valid: false, error: 'Invalid email format' }
    }
  }
  
  // GitHub username validation
  if (key === 'com.github') {
    if (!/^[a-z0-9](?:[a-z0-9]|-(?=[a-z0-9])){0,38}$/i.test(value)) {
      return { valid: false, error: 'Invalid GitHub username' }
    }
  }
  
  // General length check
  if (value.length > 1000) {
    return { valid: false, error: 'Value too long (max 1000 characters)' }
  }
  
  return { valid: true }
}

async function setTextRecordWithValidation(
  name: string,
  key: string,
  value: string
): Promise<string> {
  const validation = validateTextRecord(key, value)
  
  if (!validation.valid) {
    throw new Error(`Validation failed: ${validation.error}`)
  }
  
  return setTextRecord(name, key, value)
}

// Example with validation
try {
  await setTextRecordWithValidation('alice.base', 'com.twitter', '@invalid')
} catch (error) {
  console.error('Validation error:', error)
}

// Valid record
await setTextRecordWithValidation('alice.base', 'com.twitter', 'alice_crypto')
```

## Common Patterns

**Profile Initialization:**
```typescript
async function initializeProfile(name: string, profile: {
  twitter?: string
  github?: string
  website?: string
  bio?: string
  avatar?: string
}): Promise<void> {
  const updates: TextRecordUpdate[] = []
  
  if (profile.twitter) updates.push({ key: 'com.twitter', value: profile.twitter })
  if (profile.github) updates.push({ key: 'com.github', value: profile.github })
  if (profile.website) updates.push({ key: 'url', value: profile.website })
  if (profile.bio) updates.push({ key: 'description', value: profile.bio })
  if (profile.avatar) updates.push({ key: 'avatar', value: profile.avatar })
  
  await setMultipleTextRecords(name, updates)
}

await initializeProfile('alice.base', {
  twitter: 'alice_crypto',
  github: 'alice',
  website: 'https://alice.example.com',
  bio: 'Building on Base',
  avatar: 'ipfs://QmXxxx...'
})
```

## Gotchas

**Resolver Requirement**: You cannot set text records if the name doesn't have a resolver. Always check resolver existence first.

**Owner Permissions**: Only the name owner can set text records. Attempting to set records for a name you don't own will fail.

**Empty vs. Unset**: Setting a record to an empty string "" is different from never setting it. Empty strings are stored and returned, while unset records return "".

**Gas Costs**: Each `setText` call costs gas. For multiple updates, consider batching or using multicall patterns to reduce transaction count.

**Case Sensitivity**: Text record keys are case-sensitive. Use lowercase for standard keys (e.g., "com.twitter" not "Com.Twitter").

**Data Persistence**: Text records persist across name transfers. New owners inherit all existing records unless explicitly cleared.