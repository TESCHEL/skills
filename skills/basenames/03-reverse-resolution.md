# Reverse Resolution and Primary Names

## Overview

Reverse resolution enables addresses to resolve back to human-readable names, allowing wallets and dApps to display names instead of addresses. This lesson covers setting primary names, resolving addresses to names, and implementing reverse resolution in applications. Primary names enhance UX by showing "alice.base" instead of "0x1234..."

## Reverse Resolution Architecture

Reverse resolution uses a special subdomain structure under `addr.reverse`. Each address has a corresponding reverse record at `{address}.addr.reverse` that points to their primary name.

**Flow:**
1. Address `0x1234...` -> Reverse node `0x1234...addr.reverse`
2. Reverse node -> Primary name "alice.base"
3. Primary name -> Forward resolution back to `0x1234...`

## Setting Primary Name

Configure which basename should be the primary name for your address.

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

// Get reverse registrar from registry
const registryAbi = [
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'resolver',
    outputs: [{ name: '', type: 'address' }],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'owner',
    outputs: [{ name: '', type: 'address' }],
    stateMutability: 'view',
    type: 'function'
  }
] as const

async function getResolver(name: string): Promise<string> {
  const node = namehash(name)
  
  const resolver = await publicClient.readContract({
    address: REGISTRY,
    abi: registryAbi,
    functionName: 'resolver',
    args: [node]
  })
  
  return resolver
}

// Reverse registrar ABI
const reverseRegistrarAbi = [
  {
    inputs: [
      { name: 'owner', type: 'address' },
      { name: 'resolver', type: 'address' }
    ],
    name: 'setNameForAddr',
    outputs: [{ name: '', type: 'bytes32' }],
    stateMutability: 'nonpayable',
    type: 'function'
  },
  {
    inputs: [{ name: 'name', type: 'string' }],
    name: 'setName',
    outputs: [{ name: '', type: 'bytes32' }],
    stateMutability: 'nonpayable',
    type: 'function'
  }
] as const

// Public resolver ABI for reverse resolution
const resolverAbi = [
  {
    inputs: [{ name: 'node', type: 'bytes32' }],
    name: 'name',
    outputs: [{ name: '', type: 'string' }],
    stateMutability: 'view',
    type: 'function'
  },
  {
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'name', type: 'string' }
    ],
    name: 'setName',
    outputs: [],
    stateMutability: 'nonpayable',
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

async function setPrimaryName(name: string, ownerAddress?: string): Promise<string> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  const owner = ownerAddress || account.address
  
  // Get resolver for the name
  const resolver = await getResolver(fullName)
  
  if (resolver === '0x0000000000000000000000000000000000000000') {
    throw new Error('Name must have a resolver set')
  }
  
  // Verify ownership by checking forward resolution
  const nameNode = namehash(fullName)
  const resolvedAddress = await publicClient.readContract({
    address: resolver as `0x${string}`,
    abi: resolverAbi,
    functionName: 'addr',
    args: [nameNode]
  })
  
  if (resolvedAddress.toLowerCase() !== owner.toLowerCase()) {
    throw new Error('Name does not resolve to the specified address')
  }
  
  // Get reverse node for address
  const reverseNode = getReverseNode(owner)
  
  // Set name on reverse resolver
  const hash = await walletClient.writeContract({
    address: resolver as `0x${string}`,
    abi: resolverAbi,
    functionName: 'setName',
    args: [reverseNode, fullName]
  })
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  
  if (receipt.status === 'success') {
    return `Successfully set ${fullName} as primary name for ${owner}`
  } else {
    throw new Error('Set primary name transaction failed')
  }
}

function getReverseNode(address: string): `0x${string}` {
  // Remove 0x prefix and convert to lowercase
  const addr = address.toLowerCase().replace('0x', '')
  // Reverse domain: {addr}.addr.reverse
  const reverseName = `${addr}.addr.reverse`
  return namehash(reverseName)
}

// Example: Set alice.base as primary name
const result = await setPrimaryName('alice.base')
console.log(result)
```

## Resolving Address to Name

Reverse lookup to get the primary name for an address.

```typescript
async function getPrimaryName(address: string): Promise<string | null> {
  try {
    // Get reverse node
    const reverseNode = getReverseNode(address)
    
    // Get resolver for addr.reverse
    const reverseResolver = await getResolver('addr.reverse')
    
    if (reverseResolver === '0x0000000000000000000000000000000000000000') {
      return null
    }
    
    // Get name from reverse resolver
    const name = await publicClient.readContract({
      address: reverseResolver as `0x${string}`,
      abi: resolverAbi,
      functionName: 'name',
      args: [reverseNode]
    })
    
    if (!name) {
      return null
    }
    
    // Verify forward resolution matches (security check)
    const forwardNode = namehash(name)
    const forwardResolver = await getResolver(name)
    
    if (forwardResolver === '0x0000000000000000000000000000000000000000') {
      return null
    }
    
    const resolvedAddress = await publicClient.readContract({
      address: forwardResolver as `0x${string}`,
      abi: resolverAbi,
      functionName: 'addr',
      args: [forwardNode]
    })
    
    // Return name only if forward resolution matches
    if (resolvedAddress.toLowerCase() === address.toLowerCase()) {
      return name
    }
    
    return null
  } catch (error) {
    console.error('Error resolving primary name:', error)
    return null
  }
}

// Example: Get primary name for address
const primaryName = await getPrimaryName('0x1234567890123456789012345678901234567890')
console.log('Primary name:', primaryName || 'None set')

// Batch reverse resolution
async function getPrimaryNames(addresses: string[]): Promise<Map<string, string | null>> {
  const results = new Map<string, string | null>()
  
  for (const address of addresses) {
    const name = await getPrimaryName(address)
    results.set(address, name)
  }
  
  return results
}

const addresses = [
  '0x1234567890123456789012345678901234567890',
  '0x0987654321098765432109876543210987654321'
]

const names = await getPrimaryNames(addresses)
names.forEach((name, address) => {
  console.log(`${address}: ${name || 'No primary name'}`)
})
```

## Display Name Resolution

Create a utility function for displaying addresses with fallback to truncated address.

```typescript
function truncateAddress(address: string): string {
  if (address.length < 10) return address
  return `${address.slice(0, 6)}...${address.slice(-4)}`
}

async function getDisplayName(address: string): Promise<string> {
  const primaryName = await getPrimaryName(address)
  return primaryName || truncateAddress(address)
}

// Example: Display names in UI
const displayNames = await Promise.all([
  getDisplayName('0x1234567890123456789012345678901234567890'),
  getDisplayName('0x0987654321098765432109876543210987654321')
])

displayNames.forEach((name, index) => {
  console.log(`User ${index + 1}: ${name}`)
})

// Enhanced display with avatar
interface DisplayProfile {
  address: string
  name: string
  avatar?: string
}

async function getDisplayProfile(address: string): Promise<DisplayProfile> {
  const primaryName = await getPrimaryName(address)
  const profile: DisplayProfile = {
    address,
    name: primaryName || truncateAddress(address)
  }
  
  if (primaryName) {
    try {
      const resolver = await getResolver(primaryName)
      if (resolver !== '0x0000000000000000000000000000000000000000') {
        const node = namehash(primaryName)
        const avatar = await publicClient.readContract({
          address: resolver as `0x${string}`,
          abi: [
            {
              inputs: [
                { name: 'node', type: 'bytes32' },
                { name: 'key', type: 'string' }
              ],
              name: 'text',
              outputs: [{ name: '', type: 'string' }],
              stateMutability: 'view',
              type: 'function'
            }
          ],
          functionName: 'text',
          args: [node, 'avatar']
        })
        
        if (avatar) {
          profile.avatar = avatar
        }
      }
    } catch {
      // Avatar fetch failed, continue without it
    }
  }
  
  return profile
}

const profile = await getDisplayProfile('0x1234567890123456789012345678901234567890')
console.log('Display profile:', profile)
```

## Clearing Primary Name

Remove the primary name association for an address.

```typescript
async function clearPrimaryName(address?: string): Promise<string> {
  const owner = address || account.address
  const reverseNode = getReverseNode(owner)
  
  // Get resolver for addr.reverse
  const reverseResolver = await getResolver('addr.reverse')
  
  if (reverseResolver === '0x0000000000000000000000000000000000000000') {
    throw new Error('No reverse resolver found')
  }
  
  // Set name to empty string
  const hash = await walletClient.writeContract({
    address: reverseResolver as `0x${string}`,
    abi: resolverAbi,
    functionName: 'setName',
    args: [reverseNode, '']
  })
  
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  
  if (receipt.status === 'success') {
    return `Successfully cleared primary name for ${owner}`
  } else {
    throw new Error('Clear primary name transaction failed')
  }
}

// Example: Clear primary name
await clearPrimaryName()
```

## Integration with Smart Wallets

Set primary names during smart wallet deployment for seamless UX.

```typescript
interface SmartWalletConfig {
  owner: string
  primaryName?: string
  entryPoint: string
}

async function deploySmartWalletWithName(
  config: SmartWalletConfig
): Promise<{ walletAddress: string; primaryNameSet: boolean }> {
  const ENTRY_POINT = '0x0576a174D229E3cFA37253523E645A78A0C91B57'
  
  // Deploy smart wallet (simplified)
  // In production, use actual smart wallet factory
  const walletAddress = '0x...' // Deployed wallet address
  
  let primaryNameSet = false
  
  // Set primary name if provided
  if (config.primaryName) {
    try {
      await setPrimaryName(config.primaryName, walletAddress)
      primaryNameSet = true
    } catch (error) {
      console.error('Failed to set primary name:', error)
    }
  }
  
  return { walletAddress, primaryNameSet }
}

// Example: Deploy wallet with basename
const deployment = await deploySmartWalletWithName({
  owner: account.address,
  primaryName: 'alice.base',
  entryPoint: '0x0576a174D229E3cFA37253523E645A78A0C91B57'
})

console.log('Wallet deployed:', deployment.walletAddress)
console.log('Primary name set:', deployment.primaryNameSet)
```

## Verifying Reverse Resolution

Ensure forward and reverse resolution are consistent.

```typescript
interface ResolutionVerification {
  address: string
  primaryName: string | null
  forwardResolution: string | null
  isConsistent: boolean
}

async function verifyReverseResolution(
  address: string
): Promise<ResolutionVerification> {
  const primaryName = await getPrimaryName(address)
  
  let forwardResolution: string | null = null
  let isConsistent = false
  
  if (primaryName) {
    try {
      const resolver = await getResolver(primaryName)
      if (resolver !== '0x0000000000000000000000000000000000000000') {
        const node = namehash(primaryName)
        forwardResolution = await publicClient.readContract({
          address: resolver as `0x${string}`,
          abi: resolverAbi,
          functionName: 'addr',
          args: [node]
        })
        
        isConsistent = forwardResolution.toLowerCase() === address.toLowerCase()
      }
    } catch {
      // Forward resolution failed
    }
  }
  
  return {
    address,
    primaryName,
    forwardResolution,
    isConsistent
  }
}

// Example: Verify resolution
const verification = await verifyReverseResolution('0x1234567890123456789012345678901234567890')
console.log('Verification result:', verification)

if (!verification.isConsistent && verification.primaryName) {
  console.warn('Warning: Reverse resolution is inconsistent!')
  console.warn('Primary name:', verification.primaryName)
  console.warn('Forward resolves to:', verification.forwardResolution)
}
```

## Common Patterns

**Automatic Primary Name Setup:**
```typescript
async function ensurePrimaryName(name: string): Promise<void> {
  const fullName = name.endsWith('.base') ? name : `${name}.base`
  
  // Check if primary name is already set
  const currentPrimary = await getPrimaryName(account.address)
  
  if (currentPrimary === fullName) {
    console.log('Primary name already set correctly')
    return
  }
  
  // Set new primary name
  await setPrimaryName(fullName)
  
  // Verify it was set
  const verification = await verifyReverseResolution(account.address)
  
  if (!verification.isConsistent) {
    throw new Error('Failed to set primary name correctly')
  }
  
  console.log(`Primary name set to ${fullName}`)
}

await ensurePrimaryName('alice.base')
```

**Multi-Address Primary Names:**
```typescript
async function setPrimaryNamesForMultipleAddresses(
  assignments: Array<{ address: string; name: string }>
): Promise<Map<string, boolean>> {
  const results = new Map<string, boolean>()
  
  for (const { address, name } of assignments) {
    try {
      await setPrimaryName(name, address)
      results.set(address, true)
    } catch (error) {
      console.error(`Failed to set primary name for ${address}:`, error)
      results.set(address, false)
    }
  }
  
  return results
}

const assignments = [
  { address: '0x1234...', name: 'alice.base' },
  { address: '0x5678...', name: 'bob.base' }
]

const results = await setPrimaryNamesForMultipleAddresses(assignments)
results.forEach((success, address) => {
  console.log(`${address}: ${success ? 'Success' : 'Failed'}`)
})
```

## Gotchas

**Forward Resolution Requirement**: A name must resolve forward to your address before you can set it as your primary name. Always verify forward resolution first.

**Resolver Dependencies**: Both the forward name and the reverse record require properly configured resolvers. Missing resolvers cause silent failures.

**Security Verification**: Always verify that the forward resolution matches when reading reverse records. This prevents spoofing attacks where someone sets a primary name they don't own.

**Gas Costs**: Setting primary names requires an on-chain transaction with gas costs. Plan for this in user flows.

**Transfer Implications**: When you transfer a name to another address, the old owner's primary name association remains until explicitly cleared or changed.

**Multiple Names**: An address can only have one primary name, but can own multiple basenames. Choose wisely which name represents your primary identity.

**Case Sensitivity**: Always lowercase addresses when computing reverse nodes. Checksummed addresses will produce incorrect reverse nodes.

**Resolver Compatibility**: Ensure the resolver supports the `name()` function for reverse resolution. Not all custom resolvers implement this.