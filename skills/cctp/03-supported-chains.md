# CCTP Supported Chains

## Overview

CCTP currently supports six blockchain networks: Base, Ethereum, Arbitrum, Optimism, Polygon, and Avalanche. Each chain has unique characteristics affecting transfer costs, timing, and integration patterns. This lesson provides comprehensive details on each supported chain including domain IDs, contract addresses, fee structures, and practical considerations for cross-chain transfers.

## Domain IDs and Chain Mapping

Every CCTP-supported chain has a unique domain identifier used in cross-chain messages.

```typescript
import { Address } from 'viem';

enum CCTPDomain {
  Ethereum = 0,
  Avalanche = 1,
  Optimism = 2,
  Arbitrum = 3,
  Base = 6,
  Polygon = 7
}

interface ChainConfig {
  domain: number;
  chainId: number;
  name: string;
  rpcUrl: string;
  blockTime: number; // seconds
  finalizationTime: number; // minutes
  tokenMessenger: Address;
  messageTransmitter: Address;
  usdc: Address;
  explorer: string;
}

const CCTP_CHAIN_CONFIGS: Record<string, ChainConfig> = {
  ethereum: {
    domain: CCTPDomain.Ethereum,
    chainId: 1,
    name: 'Ethereum Mainnet',
    rpcUrl: 'https://eth.llamarpc.com',
    blockTime: 12,
    finalizationTime: 15,
    tokenMessenger: '0xBd3fa81B58Ba92a82136038B25aDec7066af3155',
    messageTransmitter: '0x0a992d191DEeC32aFe36203Ad87D7d289a738F81',
    usdc: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    explorer: 'https://etherscan.io'
  },
  avalanche: {
    domain: CCTPDomain.Avalanche,
    chainId: 43114,
    name: 'Avalanche C-Chain',
    rpcUrl: 'https://api.avax.network/ext/bc/C/rpc',
    blockTime: 2,
    finalizationTime: 10,
    tokenMessenger: '0x6B25532e1060CE10cc3B0A99e5683b91BFDe6982',
    messageTransmitter: '0x8186359aF5F57FbB40c6b14A588d2A59C0C29880',
    usdc: '0xB97EF9Ef8734C71904D8002F8b6Bc66Dd9c48a6E',
    explorer: 'https://snowtrace.io'
  },
  optimism: {
    domain: CCTPDomain.Optimism,
    chainId: 10,
    name: 'Optimism',
    rpcUrl: 'https://mainnet.optimism.io',
    blockTime: 2,
    finalizationTime: 12,
    tokenMessenger: '0x2B4069517957735bE00ceE0fadAE88a26365528f',
    messageTransmitter: '0x4D41f22c5a0e5c74090899E5a8Fb597a8842b3e8',
    usdc: '0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85',
    explorer: 'https://optimistic.etherscan.io'
  },
  arbitrum: {
    domain: CCTPDomain.Arbitrum,
    chainId: 42161,
    name: 'Arbitrum One',
    rpcUrl: 'https://arb1.arbitrum.io/rpc',
    blockTime: 0.25,
    finalizationTime: 12,
    tokenMessenger: '0x19330d10D9Cc8751218eaf51E8885D058642E08A',
    messageTransmitter: '0xC30362313FBBA5cf9163F0bb16a0e01f01A896ca',
    usdc: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831',
    explorer: 'https://arbiscan.io'
  },
  base: {
    domain: CCTPDomain.Base,
    chainId: 8453,
    name: 'Base',
    rpcUrl: 'https://mainnet.base.org',
    blockTime: 2,
    finalizationTime: 10,
    tokenMessenger: '0x1682Ae6375C4E4A97e4B583BC394c861A46D8962',
    messageTransmitter: '0xAD09780d193884d503182aD4588450C416D6F9D4',
    usdc: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
    explorer: 'https://basescan.org'
  },
  polygon: {
    domain: CCTPDomain.Polygon,
    chainId: 137,
    name: 'Polygon PoS',
    rpcUrl: 'https://polygon-rpc.com',
    blockTime: 2,
    finalizationTime: 15,
    tokenMessenger: '0x9daF8c91AEFAE50b9c0E69629D3F6Ca40cA3B3FE',
    messageTransmitter: '0xF3be9355363857F3e001be68856A2f96b4C39Ba9',
    usdc: '0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359',
    explorer: 'https://polygonscan.com'
  }
};

// Helper function to get config by chain name or domain
function getChainConfig(identifier: string | number): ChainConfig {
  if (typeof identifier === 'number') {
    const entry = Object.entries(CCTP_CHAIN_CONFIGS).find(
      ([, config]) => config.domain === identifier
    );
    if (!entry) throw new Error(`No chain found for domain ${identifier}`);
    return entry[1];
  }
  
  const config = CCTP_CHAIN_CONFIGS[identifier.toLowerCase()];
  if (!config) throw new Error(`Unsupported chain: ${identifier}`);
  return config;
}
```

## Chain-Specific Details

### Ethereum Mainnet

**Characteristics**:
- Domain ID: 0
- Longest finalization time (15-20 minutes)
- Highest gas costs
- Most liquid USDC market
- Origin chain for USDC

**Gas Considerations**:
```typescript
import { formatGwei, parseGwei } from 'viem';

async function estimateEthereumTransferCost(): Promise<{
  gasPriceGwei: string;
  burnCostUSD: string;
  mintCostUSD: string;
  totalCostUSD: string;
}> {
  // Typical gas prices on Ethereum
  const gasPrice = parseGwei('30'); // 30 gwei baseline
  const ethPriceUSD = 2000; // Approximate ETH price
  
  const burnGas = 150000n; // depositForBurn gas
  const mintGas = 180000n; // receiveMessage gas
  
  const burnCostETH = (gasPrice * burnGas) / 10n ** 18n;
  const mintCostETH = (gasPrice * mintGas) / 10n ** 18n;
  
  const burnCostUSD = Number(burnCostETH) * ethPriceUSD;
  const mintCostUSD = Number(mintCostETH) * ethPriceUSD;
  
  return {
    gasPriceGwei: formatGwei(gasPrice),
    burnCostUSD: burnCostUSD.toFixed(2),
    mintCostUSD: mintCostUSD.toFixed(2),
    totalCostUSD: (burnCostUSD + mintCostUSD).toFixed(2)
  };
}

// Example: Base to Ethereum transfer considerations
const baseToEthereumTips = {
  minTransferAmount: '50', // USD, to justify gas costs
  peakHoursAvoid: [14, 15, 16, 17], // UTC hours with high gas
  estimatedTime: '15-20 minutes',
  notes: [
    'Wait for lower gas prices during off-peak hours',
    'Consider batching multiple transfers',
    'Ethereum finalization is slower than L2s'
  ]
};
```

### Arbitrum One

**Characteristics**:
- Domain ID: 3
- Very fast block times (0.25s)
- Low gas costs
- Quick finalization (10-12 minutes)
- Large DeFi ecosystem

**Integration Example**:
```typescript
import { arbitrum } from 'viem/chains';

async function transferBaseToArbitrum(
  amount: bigint,
  recipient: Address
): Promise<void> {
  const sourceConfig = CCTP_CHAIN_CONFIGS.base;
  const destConfig = CCTP_CHAIN_CONFIGS.arbitrum;
  
  console.log('Initiating Base -> Arbitrum transfer');
  console.log(`Destination domain: ${destConfig.domain}`);
  console.log(`Destination USDC: ${destConfig.usdc}`);
  
  // Execute burn on Base
  const burnResult = await depositForBurn({
    amount,
    destinationDomain: destConfig.domain,
    mintRecipient: recipient
  });
  
  console.log(`Burned on Base: ${burnResult.transactionHash}`);
  console.log('Waiting for attestation (approx 10-12 minutes)...');
  
  const attestation = await pollForAttestation(burnResult.messageHash);
  
  // Complete on Arbitrum
  const arbitrumClient = createWalletClient({
    account,
    chain: arbitrum,
    transport: http('https://arb1.arbitrum.io/rpc')
  });
  
  const mintHash = await arbitrumClient.writeContract({
    address: destConfig.messageTransmitter,
    abi: MESSAGE_TRANSMITTER_ABI,
    functionName: 'receiveMessage',
    args: [burnResult.messageBytes, attestation as `0x${string}`]
  });
  
  console.log(`Minted on Arbitrum: ${mintHash}`);
  console.log(`View on explorer: ${destConfig.explorer}/tx/${mintHash}`);
}
```

**Arbitrum-Specific Considerations**:
```typescript
const arbitrumNotes = {
  advantages: [
    'Extremely low gas costs (< $0.10 typically)',
    'Fast block times for quick confirmations',
    'Large liquidity pools for further DeFi operations',
    'Growing NFT and gaming ecosystem'
  ],
  considerations: [
    'Native USDC replaced bridged USDC.e in June 2023',
    'Check which USDC version protocols accept',
    'Sequencer uptime dependency (centralization risk)'
  ],
  bestFor: [
    'Frequent small transfers',
    'DeFi strategies requiring low fees',
    'NFT purchases and gaming transactions'
  ]
};
```

### Optimism

**Characteristics**:
- Domain ID: 2
- Fast L2 with 2-second blocks
- Competitive gas costs
- Strong DeFi presence
- Similar finalization to Base

**Cost Comparison**:
```typescript
interface TransferCostEstimate {
  chain: string;
  burnGasCost: string;
  mintGasCost: string;
  totalUSD: string;
}

async function compareTransferCosts(
  chains: string[]
): Promise<TransferCostEstimate[]> {
  const estimates: TransferCostEstimate[] = [];
  
  const gasPrices = {
    ethereum: 30, // gwei
    base: 0.05, // gwei
    arbitrum: 0.1, // gwei
    optimism: 0.05, // gwei
    polygon: 50, // gwei
    avalanche: 25 // gwei
  };
  
  const nativePrices = {
    ethereum: 2000,
    base: 2000, // Uses ETH
    arbitrum: 2000, // Uses ETH
    optimism: 2000, // Uses ETH
    polygon: 0.60,
    avalanche: 20
  };
  
  for (const chain of chains) {
    const gasPrice = gasPrices[chain as keyof typeof gasPrices];
    const nativePrice = nativePrices[chain as keyof typeof nativePrices];
    
    const burnGasUnits = 150000;
    const mintGasUnits = 180000;
    
    const burnCostNative = (gasPrice * burnGasUnits) / 1e9;
    const mintCostNative = (gasPrice * mintGasUnits) / 1e9;
    
    const burnCostUSD = burnCostNative * nativePrice;
    const mintCostUSD = mintCostNative * nativePrice;
    
    estimates.push({
      chain,
      burnGasCost: `$${burnCostUSD.toFixed(2)}`,
      mintGasCost: `$${mintCostUSD.toFixed(2)}`,
      totalUSD: `$${(burnCostUSD + mintCostUSD).toFixed(2)}`
    });
  }
  
  return estimates;
}

// Usage
const costComparison = await compareTransferCosts([
  'ethereum',
  'base',
  'arbitrum',
  'optimism'
]);

console.table(costComparison);
// Expected output:
// ethereum: ~$9-15 total
// base: ~$0.06 total
// arbitrum: ~$0.10 total
// optimism: ~$0.06 total
```

### Polygon PoS

**Characteristics**:
- Domain ID: 7
- Independent PoS chain (not L2)
- Very low gas costs (paid in MATIC)
- Longer finalization (15-18 minutes)
- Large user base and DeFi ecosystem

**Polygon-Specific Implementation**:
```typescript
import { polygon } from 'viem/chains';

async function handlePolygonTransfer(
  direction: 'to' | 'from',
  amount: bigint,
  address: Address
): Promise<void> {
  const polygonConfig = CCTP_CHAIN_CONFIGS.polygon;
  const baseConfig = CCTP_CHAIN_CONFIGS.base;
  
  if (direction === 'to') {
    // Base -> Polygon
    console.log('Transferring from Base to Polygon');
    console.log('Note: Polygon uses MATIC for gas, ensure recipient has MATIC');
    
    await depositForBurn({
      amount,
      destinationDomain: polygonConfig.domain,
      mintRecipient: address
    });
  } else {
    // Polygon -> Base
    console.log('Transferring from Polygon to Base');
    
    const polygonClient = createWalletClient({
      account,
      chain: polygon,
      transport: http(polygonConfig.rpcUrl)
    });
    
    // Burn on Polygon
    const hash = await polygonClient.writeContract({
      address: polygonConfig.tokenMessenger,
      abi: TOKEN_MESSENGER_ABI,
      functionName: 'depositForBurn',
      args: [
        amount,
        baseConfig.domain,
        addressToBytes32(address),
        polygonConfig.usdc
      ]
    });
    
    console.log(`Burned on Polygon: ${hash}`);
  }
}
```

**Polygon Considerations**:
```typescript
const polygonIntegrationNotes = {
  gasToken: 'MATIC (not ETH)',
  gasCosts: 'Very low, typically < $0.01',
  finalization: '15-18 minutes (checkpoint mechanism)',
  usdcAddress: '0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359',
  warnings: [
    'Recipients need MATIC for gas to interact with received USDC',
    'Checkpoint validators can occasionally delay finality',
    'Bridge congestion during high activity'
  ],
  advantages: [
    'Massive user base and liquidity',
    'Extensive DeFi protocol support',
    'Very low transaction costs',
    'Strong gaming and NFT ecosystem'
  ]
};
```

### Avalanche C-Chain

**Characteristics**:
- Domain ID: 1
- Fast finality (1-2 seconds)
- Moderate gas costs (paid in AVAX)
- Quick attestation (10-12 minutes)
- Growing DeFi ecosystem

**Avalanche Integration**:
```typescript
import { avalanche } from 'viem/chains';

const avalancheFeatures = {
  subnetSupport: 'Can extend to custom subnets',
  consensusSpeed: 'Sub-second finality',
  gasToken: 'AVAX',
  typicalGasCost: '$0.10-0.50',
  usdcType: 'Native USDC via CCTP',
  ecosystem: [
    'TraderJoe DEX',
    'Aave lending',
    'Benqi liquid staking',
    'Growing gaming sector'
  ]
};

// Example: Base to Avalanche with gas estimation
async function transferToAvalancheWithEstimate(
  amount: bigint,
  recipient: Address
): Promise<{
  estimatedGasCost: string;
  estimatedTime: string;
  transactionHash: Address;
}> {
  const avaxPrice = 25; // USD
  const typicalGasPrice = 25; // nAVAX
  const burnGas = 150000;
  
  const gasCostAVAX = (typicalGasPrice * burnGas) / 1e9;
  const gasCostUSD = gasCostAVAX * avaxPrice;
  
  const result = await depositForBurn({
    amount,
    destinationDomain: CCTP_CHAIN_CONFIGS.avalanche.domain,
    mintRecipient: recipient
  });
  
  return {
    estimatedGasCost: `$${gasCostUSD.toFixed(2)}`,
    estimatedTime: '10-12 minutes',
    transactionHash: result.transactionHash
  };
}
```

### Base (Source Chain Context)

**Characteristics**:
- Domain ID: 6
- Coinbase L2 (OP Stack)
- Extremely low gas costs
- Fast finalization (10-15 minutes)
- Native integration with Coinbase

**Base as CCTP Hub**:
```typescript
const baseAdvantages = {
  gasEfficiency: 'Lowest cost L2',
  coinbaseIntegration: 'Direct fiat on/off ramps',
  finalization: 'Faster than Ethereum L1',
  ecosystem: 'Rapidly growing DeFi and consumer apps',
  cctp: 'Full CCTP support from launch',
  recommendations: [
    'Ideal source chain for CCTP transfers',
    'Aggregate funds on Base before bridging',
    'Use for frequent cross-chain operations',
    'Leverage Coinbase ecosystem integration'
  ]
};

// Multi-destination transfer from Base
async function distributeFundsFromBase(
  destinations: Array<{
    chain: string;
    recipient: Address;
    amount: bigint;
  }>
): Promise<void> {
  console.log(`Distributing USDC from Base to ${destinations.length} chains`);
  
  for (const dest of destinations) {
    const config = getChainConfig(dest.chain);
    
    console.log(`\nTransferring to ${config.name}...`);
    console.log(`Amount: ${dest.amount} USDC`);
    console.log(`Recipient: ${dest.recipient}`);
    
    const result = await depositForBurn({
      amount: dest.amount,
      destinationDomain: config.domain,
      mintRecipient: dest.recipient
    });
    
    console.log(`Burned on Base: ${result.transactionHash}`);
    console.log(`Message hash: ${result.messageHash}`);
    console.log(`Estimated completion: ${config.finalizationTime} minutes`);
  }
}
```

## Fee Structures and Optimization

### Per-Chain Fee Breakdown

```typescript
interface ChainFeeStructure {
  burnGasUnits: number;
  mintGasUnits: number;
  typicalGasPrice: string;
  nativeToken: string;
  bridgeFee: number; // CCTP has no protocol fee
  totalCostRange: string;
}

const chainFees: Record<string, ChainFeeStructure> = {
  ethereum: {
    burnGasUnits: 150000,
    mintGasUnits: 180000,
    typicalGasPrice: '20-50 gwei',
    nativeToken: 'ETH',
    bridgeFee: 0,
    totalCostRange: '$6-20'
  },
  base: {
    burnGasUnits: 150000,
    mintGasUnits: 180000,
    typicalGasPrice: '0.01-0.1 gwei',
    nativeToken: 'ETH',
    bridgeFee: 0,
    totalCostRange: '$0.02-0.20'
  },
  arbitrum: {
    burnGasUnits: 150000,
    mintGasUnits: 180000,
    typicalGasPrice: '0.1-0.5 gwei',
    nativeToken: 'ETH',
    bridgeFee: 0,
    totalCostRange: '$0.05-0.30'
  },
  optimism: {
    burnGasUnits: 150000,
    mintGasUnits: 180000,
    typicalGasPrice: '0.01-0.1 gwei',
    nativeToken: 'ETH',
    bridgeFee: 0,
    totalCostRange: '$0.02-0.20'
  },
  polygon: {
    burnGasUnits: 150000,
    mintGasUnits: 180000,
    typicalGasPrice: '30-100 gwei',
    nativeToken: 'MATIC',
    bridgeFee: 0,
    totalCostRange: '$0.01-0.05'
  },
  avalanche: {
    burnGasUnits: 150000,
    mintGasUnits: 180000,
    typicalGasPrice: '25-50 nAVAX',
    nativeToken: 'AVAX',
    bridgeFee: 0,
    totalCostRange: '$0.10-0.50'
  }
};
```

### Optimization Strategies

```typescript
// Choose optimal route based on cost and time
function selectOptimalRoute(
  sourceChain: string,
  destinationChain: string,
  transferAmount: bigint,
  urgency: 'low' | 'medium' | 'high'
): {
  recommended: boolean;
  reason: string;
  alternativeRoute?: string;
} {
  const source = getChainConfig(sourceChain);
  const dest = getChainConfig(destinationChain);
  
  // Direct transfer is always optimal for CCTP
  if (urgency === 'high' && dest.finalizationTime > 15) {
    return {
      recommended: false,
      reason: `${dest.name} finalization takes ${dest.finalizationTime} minutes`,
      alternativeRoute: 'Consider faster destination or alternative bridge'
    };
  }
  
  // Check if amount justifies gas costs
  const minAmount = source.name === 'Ethereum' ? 50n * 10n ** 6n : 10n * 10n ** 6n;
  if (transferAmount < minAmount) {
    return {
      recommended: false,
      reason: `Transfer amount too small for ${source.name} gas costs`,
      alternativeRoute: 'Batch multiple transfers or use lower-cost source chain'
    };
  }
  
  return {
    recommended: true,
    reason: 'Optimal CCTP route selected'
  };
}

// Batch transfer optimization
async function batchTransfersOptimally(
  transfers: Array<{
    destination: string;
    recipient: Address;
    amount: bigint;
  }>
): Promise<void> {
  // Group by destination chain
  const groupedByChain = transfers.reduce((acc, transfer) => {
    if (!acc[transfer.destination]) {
      acc[transfer.destination] = [];
    }
    acc[transfer.destination].push(transfer);
    return acc;
  }, {} as Record<string, typeof transfers>);
  
  // Process each destination
  for (const [chainName, chainTransfers] of Object.entries(groupedByChain)) {
    console.log(`\nProcessing ${chainTransfers.length} transfers to ${chainName}`);
    
    // Could consolidate to single recipient if appropriate
    for (const transfer of chainTransfers) {
      await depositForBurn({
        amount: transfer.amount,
        destinationDomain: getChainConfig(chainName).domain,
        mintRecipient: transfer.recipient
      });
      
      // Small delay to avoid nonce issues
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
  }
}
```

## Best Practices by Use Case

```typescript
const useCaseRecommendations = {
  deFiArbitrage: {
    preferredChains: ['base', 'arbitrum'],
    reason: 'Lowest fees enable frequent rebalancing',
    pattern: 'High frequency, smaller amounts'
  },
  
  treasuryManagement: {
    preferredChains: ['ethereum', 'base'],
    reason: 'Maximum security and liquidity',
    pattern: 'Infrequent, larger amounts'
  },
  
  userPayments: {
    preferredChains: ['base', 'polygon', 'arbitrum'],
    reason: 'Low costs improve user experience',
    pattern: 'Variable frequency and amounts'
  },
  
  nftPurchases: {
    preferredChains: ['base', 'arbitrum', 'polygon'],
    reason: 'Ecosystems with strong NFT markets',
    pattern: 'Sporadic, medium amounts'
  },
  
  crossChainYield: {
    preferredChains: ['arbitrum', 'optimism', 'avalanche'],
    reason: 'Diverse DeFi protocols with competitive yields',
    pattern: 'Periodic rebalancing, larger amounts'
  }
};

// Implement use case specific logic
function getRecommendedChainForUseCase(
  useCase: keyof typeof useCaseRecommendations,
  currentChain: string
): string[] {
  const rec = useCaseRecommendations[useCase];
  return rec.preferredChains.filter(chain => chain !== currentChain);
}
```