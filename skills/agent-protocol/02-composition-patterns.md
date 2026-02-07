# Agent Protocol Composition Patterns

## Overview

This lesson covers three fundamental composition patterns for building agent commerce systems: agent registration with payment setup, service discovery with attestation verification, and cross-chain agent commerce. Each pattern combines multiple stack components into reusable flows.

## Pattern 1: Agent Registration + Payment Setup

This pattern onboards a new agent into the ecosystem with full identity, payment, and discovery capabilities.

```typescript
import { createPublicClient, createWalletClient, http, encodeFunctionData } from 'viem';
import { base } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';
import { initiateUserControlledWalletsClient } from '@circle-fin/user-controlled-wallets';

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432';
const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63';
const EAS_CONTRACT = '0x4200000000000000000000000000000000000021';
const BASENAME_REGISTRAR = '0x4cCb0720c37C1658477b9C7A3aEBb87f1a067292';
const ENTRYPOINT = '0x0576a174D229E3cFA37253523E645A78A0C91B57';

interface AgentRegistrationParams {
  name: string;
  description: string;
  capabilities: string[];
  serviceEndpoint: string;
  pricingModel: {
    currency: 'USDC';
    basePrice: string;
    unit: 'per-request' | 'per-hour' | 'per-token';
  };
}

class AgentOnboardingService {
  private publicClient;
  private walletClient;
  private circleClient;
  
  constructor(privateKey: string, circleApiKey: string) {
    const account = privateKeyToAccount(privateKey as `0x${string}`);
    
    this.publicClient = createPublicClient({
      chain: base,
      transport: http('https://mainnet.base.org')
    });
    
    this.walletClient = createWalletClient({
      account,
      chain: base,
      transport: http('https://mainnet.base.org')
    });
    
    this.circleClient = initiateUserControlledWalletsClient({
      apiKey: circleApiKey
    });
  }
  
  async registerAgent(params: AgentRegistrationParams) {
    console.log('Step 1: Deploying smart wallet...');
    const smartWallet = await this.deploySmartWallet();
    
    console.log('Step 2: Creating Circle wallet...');
    const circleWallet = await this.createCircleWallet(smartWallet.address, params.name);
    
    console.log('Step 3: Uploading metadata to IPFS...');
    const metadataURI = await this.uploadMetadata({
      name: params.name,
      description: params.description,
      serviceEndpoint: params.serviceEndpoint,
      pricingModel: params.pricingModel,
      wallets: {
        smart: smartWallet.address,
        circle: circleWallet.address
      }
    });
    
    console.log('Step 4: Registering identity...');
    const agentId = await this.registerIdentity(
      smartWallet.address,
      metadataURI,
      params.capabilities
    );
    
    console.log('Step 5: Creating capability attestations...');
    const attestations = await this.createCapabilityAttestations(
      smartWallet.address,
      params.capabilities
    );
    
    console.log('Step 6: Registering basename...');
    const basename = await this.registerBasename(
      params.name + '.base.eth',
      smartWallet.address
    );
    
    return {
      agentId,
      smartWallet: smartWallet.address,
      circleWallet: circleWallet.address,
      basename,
      metadataURI,
      attestations
    };
  }
  
  private async deploySmartWallet() {
    const factoryAbi = [{
      name: 'createAccount',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'owner', type: 'address' },
        { name: 'salt', type: 'uint256' }
      ],
      outputs: [{ name: 'account', type: 'address' }]
    }];
    
    const factoryAddress = '0x...';
    const salt = BigInt(Date.now());
    
    const hash = await this.walletClient.writeContract({
      address: factoryAddress,
      abi: factoryAbi,
      functionName: 'createAccount',
      args: [this.walletClient.account.address, salt]
    });
    
    const receipt = await this.publicClient.waitForTransactionReceipt({ hash });
    const walletAddress = receipt.logs[0].address;
    
    return { address: walletAddress, salt };
  }
  
  private async createCircleWallet(smartWalletAddress: string, agentName: string) {
    const response = await this.circleClient.createWallet({
      idempotencyKey: 'agent-' + smartWalletAddress.toLowerCase(),
      accountType: 'SCA',
      blockchains: ['BASE'],
      metadata: [
        { name: 'agentName', value: agentName },
        { name: 'smartWallet', value: smartWalletAddress },
        { name: 'createdAt', value: new Date().toISOString() }
      ]
    });
    
    return {
      id: response.data.wallet.id,
      address: response.data.wallet.address
    };
  }
  
  private async uploadMetadata(metadata: any): Promise<string> {
    const response = await fetch('https://api.pinata.cloud/pinning/pinJSONToIPFS', {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer ' + process.env.PINATA_JWT,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        pinataContent: metadata,
        pinataMetadata: {
          name: 'agent-metadata-' + Date.now()
        }
      })
    });
    
    const result = await response.json();
    return 'ipfs://' + result.IpfsHash;
  }
  
  private async registerIdentity(
    agentAddress: string,
    metadataURI: string,
    capabilities: string[]
  ): Promise<bigint> {
    const identityAbi = [{
      name: 'register',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'agentAddress', type: 'address' },
        { name: 'metadataURI', type: 'string' },
        { name: 'capabilities', type: 'bytes32[]' }
      ],
      outputs: [{ name: 'agentId', type: 'uint256' }]
    }];
    
    const capabilityHashes = capabilities.map(cap => 
      '0x' + Buffer.from(cap).toString('hex').padEnd(64, '0')
    );
    
    const hash = await this.walletClient.writeContract({
      address: IDENTITY_REGISTRY,
      abi: identityAbi,
      functionName: 'register',
      args: [agentAddress, metadataURI, capabilityHashes]
    });
    
    const receipt = await this.publicClient.waitForTransactionReceipt({ hash });
    return BigInt(receipt.logs[0].topics[1] || '0');
  }
  
  private async createCapabilityAttestations(
    agentAddress: string,
    capabilities: string[]
  ): Promise<string[]> {
    const easAbi = [{
      name: 'attest',
      type: 'function',
      stateMutability: 'payable',
      inputs: [{
        name: 'request',
        type: 'tuple',
        components: [
          { name: 'schema', type: 'bytes32' },
          { name: 'data', type: 'tuple', components: [
            { name: 'recipient', type: 'address' },
            { name: 'expirationTime', type: 'uint64' },
            { name: 'revocable', type: 'bool' },
            { name: 'refUID', type: 'bytes32' },
            { name: 'data', type: 'bytes' },
            { name: 'value', type: 'uint256' }
          ]}
        ]
      }],
      outputs: [{ name: 'uid', type: 'bytes32' }]
    }];
    
    const capabilitySchema = '0x1234...'; // Capability attestation schema
    const attestationUIDs: string[] = [];
    
    for (const capability of capabilities) {
      const attestationData = encodeFunctionData({
        abi: [{
          name: 'encode',
          type: 'function',
          inputs: [
            { name: 'capability', type: 'string' },
            { name: 'verified', type: 'bool' },
            { name: 'timestamp', type: 'uint64' }
          ],
          outputs: [{ name: '', type: 'bytes' }]
        }],
        functionName: 'encode',
        args: [capability, true, BigInt(Math.floor(Date.now() / 1000))]
      });
      
      const hash = await this.walletClient.writeContract({
        address: EAS_CONTRACT,
        abi: easAbi,
        functionName: 'attest',
        args: [{
          schema: capabilitySchema,
          data: {
            recipient: agentAddress,
            expirationTime: BigInt(0),
            revocable: true,
            refUID: '0x0000000000000000000000000000000000000000000000000000000000000000',
            data: attestationData,
            value: BigInt(0)
          }
        }]
      });
      
      const receipt = await this.publicClient.waitForTransactionReceipt({ hash });
      attestationUIDs.push(receipt.logs[0].topics[1] || '');
    }
    
    return attestationUIDs;
  }
  
  private async registerBasename(name: string, address: string): Promise<string> {
    const registrarAbi = [{
      name: 'register',
      type: 'function',
      stateMutability: 'payable',
      inputs: [
        { name: 'name', type: 'string' },
        { name: 'owner', type: 'address' },
        { name: 'duration', type: 'uint256' },
        { name: 'resolver', type: 'address' },
        { name: 'data', type: 'bytes[]' },
        { name: 'reverseRecord', type: 'bool' }
      ],
      outputs: []
    }];
    
    const hash = await this.walletClient.writeContract({
      address: BASENAME_REGISTRAR,
      abi: registrarAbi,
      functionName: 'register',
      args: [
        name.replace('.base.eth', ''),
        address,
        BigInt(31536000), // 1 year
        '0x...', // Default resolver
        [],
        true
      ],
      value: BigInt('1000000000000000') // 0.001 ETH
    });
    
    await this.publicClient.waitForTransactionReceipt({ hash });
    return name;
  }
}

// Usage
const onboarding = new AgentOnboardingService(
  process.env.PRIVATE_KEY!,
  process.env.CIRCLE_API_KEY!
);

const result = await onboarding.registerAgent({
  name: 'contract-analyzer',
  description: 'AI agent specializing in smart contract security analysis',
  capabilities: ['smart-contract-audit', 'vulnerability-detection', 'gas-optimization'],
  serviceEndpoint: 'https://api.example.com/analyze',
  pricingModel: {
    currency: 'USDC',
    basePrice: '5000000', // 5 USDC
    unit: 'per-request'
  }
});

console.log('Agent registered:', result);
```

## Pattern 2: Service Discovery + Attestation Verification

This pattern discovers agents by capability and verifies their reputation through attestations.

```typescript
class AgentDiscoveryService {
  private publicClient;
  
  constructor() {
    this.publicClient = createPublicClient({
      chain: base,
      transport: http('https://mainnet.base.org')
    });
  }
  
  async discoverAgents(capability: string, minReputation: number = 80) {
    console.log('Step 1: Querying identity registry...');
    const agents = await this.queryAgentsByCapability(capability);
    
    console.log('Step 2: Fetching metadata...');
    const agentsWithMetadata = await Promise.all(
      agents.map(async (agent) => {
        const metadata = await this.fetchMetadata(agent.metadataURI);
        return { ...agent, metadata };
      })
    );
    
    console.log('Step 3: Verifying attestations...');
    const agentsWithAttestations = await Promise.all(
      agentsWithMetadata.map(async (agent) => {
        const attestations = await this.fetchAttestations(agent.address);
        const verifiedAttestations = await this.verifyAttestations(attestations);
        return { ...agent, attestations: verifiedAttestations };
      })
    );
    
    console.log('Step 4: Calculating reputation scores...');
    const agentsWithScores = await Promise.all(
      agentsWithAttestations.map(async (agent) => {
        const score = await this.calculateReputationScore(agent.address);
        return { ...agent, reputationScore: score };
      })
    );
    
    console.log('Step 5: Filtering by reputation...');
    const qualifiedAgents = agentsWithScores.filter(
      agent => agent.reputationScore >= minReputation
    );
    
    return qualifiedAgents.sort((a, b) => b.reputationScore - a.reputationScore);
  }
  
  private async queryAgentsByCapability(capability: string) {
    const registryAbi = [{
      name: 'getAgentsByCapability',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'capability', type: 'bytes32' }],
      outputs: [{
        name: 'agents',
        type: 'tuple[]',
        components: [
          { name: 'agentId', type: 'uint256' },
          { name: 'address', type: 'address' },
          { name: 'metadataURI', type: 'string' },
          { name: 'registeredAt', type: 'uint256' }
        ]
      }]
    }];
    
    const capabilityHash = '0x' + Buffer.from(capability).toString('hex').padEnd(64, '0');
    
    const agents = await this.publicClient.readContract({
      address: IDENTITY_REGISTRY,
      abi: registryAbi,
      functionName: 'getAgentsByCapability',
      args: [capabilityHash]
    });
    
    return agents as any[];
  }
  
  private async fetchMetadata(uri: string) {
    if (uri.startsWith('ipfs://')) {
      const cid = uri.replace('ipfs://', '');
      const response = await fetch('https://ipfs.io/ipfs/' + cid);
      return await response.json();
    }
    const response = await fetch(uri);
    return await response.json();
  }
  
  private async fetchAttestations(agentAddress: string) {
    const response = await fetch(
      'https://base.easscan.org/graphql',
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          query: `
            query Attestations($recipient: String!) {
              attestations(where: { recipient: { equals: $recipient } }) {
                id
                attester
                recipient
                refUID
                revocable
                revocationTime
                expirationTime
                data
                schema {
                  id
                }
              }
            }
          `,
          variables: { recipient: agentAddress.toLowerCase() }
        })
      }
    );
    
    const result = await response.json();
    return result.data.attestations;
  }
  
  private async verifyAttestations(attestations: any[]) {
    const easAbi = [{
      name: 'getAttestation',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'uid', type: 'bytes32' }],
      outputs: [{
        name: 'attestation',
        type: 'tuple',
        components: [
          { name: 'uid', type: 'bytes32' },
          { name: 'schema', type: 'bytes32' },
          { name: 'time', type: 'uint64' },
          { name: 'expirationTime', type: 'uint64' },
          { name: 'revocationTime', type: 'uint64' },
          { name: 'refUID', type: 'bytes32' },
          { name: 'recipient', type: 'address' },
          { name: 'attester', type: 'address' },
          { name: 'revocable', type: 'bool' },
          { name: 'data', type: 'bytes' }
        ]
      }]
    }];
    
    const verified = [];
    
    for (const att of attestations) {
      const onchain = await this.publicClient.readContract({
        address: EAS_CONTRACT,
        abi: easAbi,
        functionName: 'getAttestation',
        args: [att.id]
      });
      
      if (onchain.revocationTime === BigInt(0)) {
        verified.push({ ...att, valid: true });
      }
    }
    
    return verified;
  }
  
  private async calculateReputationScore(agentAddress: string): Promise<number> {
    const reputationAbi = [{
      name: 'getReputation',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'agent', type: 'address' }],
      outputs: [
        { name: 'score', type: 'uint256' },
        { name: 'totalServices', type: 'uint256' },
        { name: 'successfulServices', type: 'uint256' }
      ]
    }];
    
    const reputation = await this.publicClient.readContract({
      address: REPUTATION_REGISTRY,
      abi: reputationAbi,
      functionName: 'getReputation',
      args: [agentAddress]
    });
    
    return Number(reputation[0]);
  }
}

// Usage
const discovery = new AgentDiscoveryService();
const agents = await discovery.discoverAgents('smart-contract-audit', 85);
console.log('Found qualified agents:', agents);
```

## Pattern 3: Cross-Chain Agent Commerce

This pattern enables agents to operate across multiple chains using CCTP for payment bridging.

```typescript
const CCTP_MESSENGER_BASE = '0x1682Ae6375C4E4A97e4B583BC394c861A46D8962';
const CCTP_MESSENGER_ETH = '0xBd3fa81B58Ba92a82136038B25aDec7066af3155';
const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const USDC_ETH = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

class CrossChainAgentService {
  private baseClient;
  private ethClient;
  
  constructor() {
    this.baseClient = createPublicClient({
      chain: base,
      transport: http('https://mainnet.base.org')
    });
    
    this.ethClient = createPublicClient({
      chain: { id: 1, name: 'Ethereum' } as any,
      transport: http('https://eth.llamarpc.com')
    });
  }
  
  async executeCrossChainService(
    sourceChain: 'ethereum' | 'base',
    agentAddress: string,
    serviceParams: any,
    paymentAmount: string
  ) {
    if (sourceChain === 'ethereum') {
      console.log('Step 1: Bridging USDC from Ethereum to Base...');
      const bridgeTx = await this.bridgeFromEthereum(agentAddress, paymentAmount);
      
      console.log('Step 2: Waiting for bridge confirmation...');
      await this.waitForBridgeCompletion(bridgeTx.attestation);
      
      console.log('Step 3: Executing service on Base...');
      const result = await this.executeServiceOnBase(agentAddress, serviceParams);
      
      return result;
    } else {
      console.log('Step 1: Bridging USDC from Base to Ethereum...');
      const bridgeTx = await this.bridgeFromBase(agentAddress, paymentAmount);
      
      console.log('Step 2: Executing service...');
      const result = await this.executeService(agentAddress, serviceParams);
      
      return result;
    }
  }
  
  private async bridgeFromEthereum(recipient: string, amount: string) {
    const cctpAbi = [{
      name: 'depositForBurn',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'amount', type: 'uint256' },
        { name: 'destinationDomain', type: 'uint32' },
        { name: 'mintRecipient', type: 'bytes32' },
        { name: 'burnToken', type: 'address' }
      ],
      outputs: [{ name: 'nonce', type: 'uint64' }]
    }];
    
    const mintRecipient = '0x' + recipient.slice(2).padStart(64, '0');
    
    // This would be executed by the wallet client on Ethereum
    return {
      txHash: '0x...',
      nonce: BigInt(12345),
      attestation: 'pending'
    };
  }
  
  private async waitForBridgeCompletion(attestation: string): Promise<void> {
    // Poll Circle attestation service
    const maxAttempts = 60;
    let attempts = 0;
    
    while (attempts < maxAttempts) {
      const response = await fetch(
        'https://iris-api.circle.com/attestations/' + attestation
      );
      
      if (response.ok) {
        const data = await response.json();
        if (data.status === 'complete') {
          return;
        }
      }
      
      await new Promise(resolve => setTimeout(resolve, 10000)); // 10 seconds
      attempts++;
    }
    
    throw new Error('Bridge completion timeout');
  }
  
  private async executeServiceOnBase(agentAddress: string, params: any) {
    // Service execution logic
    return { success: true, result: params };
  }
}
```

## Common Patterns

**Batch Operations**: Use ERC-4337 to batch identity registration, attestation creation, and payment setup into single user operation.

**Reputation-Weighted Discovery**: Sort agents by reputation score combined with attestation recency and count.

**Payment Escrow**: Hold USDC in smart contract until service delivery attestation is created.

## Gotchas

**Metadata Immutability**: IPFS URIs are immutable. Update patterns require new registry transactions.

**Attestation Spam**: Implement attestation filtering by attester reputation to prevent spam.

**Cross-Chain Timing**: Always implement timeout and retry logic for CCTP bridges.