# Complete Agent Commerce Flow

## Overview

This lesson covers the complete end-to-end flow for agent commerce: discovering an agent, verifying attestations, negotiating payment via HTTP 402, executing gasless USDC transfer with ERC-3009, receiving service, and creating a delivery attestation. This is the canonical implementation of the BAiSED Agent Protocol.

## HTTP 402 Payment Protocol

The HTTP 402 Payment Required status enables standardized payment challenges and proofs.

```typescript
import express from 'express';
import { createPublicClient, http, recoverMessageAddress, keccak256, toBytes } from 'viem';
import { base } from 'viem/chains';

const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';

interface PaymentChallenge {
  method: 'ERC-3009';
  token: string;
  amount: string;
  recipient: string;
  validAfter: number;
  validBefore: number;
  nonce: string;
  facilitator: string;
}

interface PaymentProof {
  from: string;
  to: string;
  value: string;
  validAfter: string;
  validBefore: string;
  nonce: string;
  signature: string;
}

class AgentServiceProvider {
  private app;
  private publicClient;
  private usedNonces: Set<string>;
  
  constructor() {
    this.app = express();
    this.app.use(express.json());
    this.usedNonces = new Set();
    
    this.publicClient = createPublicClient({
      chain: base,
      transport: http('https://mainnet.base.org')
    });
    
    this.setupRoutes();
  }
  
  private setupRoutes() {
    // Service endpoint protected by HTTP 402
    this.app.post('/api/analyze-contract', async (req, res) => {
      const authHeader = req.headers.authorization;
      
      if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return this.sendPaymentChallenge(res, '1000000'); // 1 USDC
      }
      
      try {
        const paymentProof = this.decodePaymentProof(authHeader);
        const isValid = await this.verifyPaymentProof(paymentProof);
        
        if (!isValid) {
          return res.status(402).json({
            error: 'Invalid payment proof',
            details: 'Signature verification failed or payment already used'
          });
        }
        
        // Execute payment transfer
        await this.executeTransfer(paymentProof);
        
        // Mark nonce as used
        this.usedNonces.add(paymentProof.nonce);
        
        // Execute service
        const result = await this.analyzeContract(req.body.contractAddress);
        
        // Create service delivery attestation
        const attestationUID = await this.createDeliveryAttestation(
          paymentProof.from,
          req.body.contractAddress,
          result
        );
        
        res.json({
          analysis: result,
          attestation: attestationUID,
          transactionHash: '0x...'
        });
      } catch (error) {
        res.status(500).json({ error: 'Service execution failed' });
      }
    });
    
    // Payment facilitator endpoint
    this.app.post('/api/execute-transfer', async (req, res) => {
      try {
        const { paymentProof } = req.body;
        const isValid = await this.verifyPaymentProof(paymentProof);
        
        if (!isValid) {
          return res.status(400).json({ error: 'Invalid payment proof' });
        }
        
        const txHash = await this.executeTransfer(paymentProof);
        this.usedNonces.add(paymentProof.nonce);
        
        res.json({ transactionHash: txHash });
      } catch (error) {
        res.status(500).json({ error: 'Transfer execution failed' });
      }
    });
  }
  
  private sendPaymentChallenge(res: express.Response, amount: string) {
    const challenge: PaymentChallenge = {
      method: 'ERC-3009',
      token: USDC_BASE,
      amount: amount,
      recipient: '0xServiceProviderAddress...',
      validAfter: Math.floor(Date.now() / 1000),
      validBefore: Math.floor(Date.now() / 1000) + 3600,
      nonce: this.generateNonce(),
      facilitator: '0xFacilitatorAddress...'
    };
    
    res.status(402).json({
      error: 'Payment Required',
      payment: challenge,
      instructions: {
        step1: 'Sign ERC-3009 transferWithAuthorization message',
        step2: 'Submit signature as Bearer token in Authorization header',
        step3: 'Facilitator will execute transfer on your behalf'
      }
    });
  }
  
  private decodePaymentProof(authHeader: string): PaymentProof {
    const token = authHeader.split(' ')[1];
    const decoded = Buffer.from(token, 'base64').toString();
    return JSON.parse(decoded);
  }
  
  private async verifyPaymentProof(proof: PaymentProof): Promise<boolean> {
    // Check nonce not already used
    if (this.usedNonces.has(proof.nonce)) {
      return false;
    }
    
    // Check validity window
    const now = Math.floor(Date.now() / 1000);
    const validAfter = parseInt(proof.validAfter);
    const validBefore = parseInt(proof.validBefore);
    
    if (now < validAfter || now > validBefore) {
      return false;
    }
    
    // Verify ERC-3009 signature
    const domain = {
      name: 'USD Coin',
      version: '2',
      chainId: 8453,
      verifyingContract: USDC_BASE
    };
    
    const types = {
      TransferWithAuthorization: [
        { name: 'from', type: 'address' },
        { name: 'to', type: 'address' },
        { name: 'value', type: 'uint256' },
        { name: 'validAfter', type: 'uint256' },
        { name: 'validBefore', type: 'uint256' },
        { name: 'nonce', type: 'bytes32' }
      ]
    };
    
    const message = {
      from: proof.from,
      to: proof.to,
      value: BigInt(proof.value),
      validAfter: BigInt(proof.validAfter),
      validBefore: BigInt(proof.validBefore),
      nonce: proof.nonce
    };
    
    try {
      const recoveredAddress = await recoverMessageAddress({
        message: JSON.stringify(message),
        signature: proof.signature as `0x${string}`
      });
      
      return recoveredAddress.toLowerCase() === proof.from.toLowerCase();
    } catch (error) {
      return false;
    }
  }
  
  private async executeTransfer(proof: PaymentProof): Promise<string> {
    const usdcAbi = [{
      name: 'transferWithAuthorization',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'from', type: 'address' },
        { name: 'to', type: 'address' },
        { name: 'value', type: 'uint256' },
        { name: 'validAfter', type: 'uint256' },
        { name: 'validBefore', type: 'uint256' },
        { name: 'nonce', type: 'bytes32' },
        { name: 'v', type: 'uint8' },
        { name: 'r', type: 'bytes32' },
        { name: 's', type: 'bytes32' }
      ],
      outputs: []
    }];
    
    // Parse signature into v, r, s
    const sig = proof.signature.slice(2);
    const r = '0x' + sig.slice(0, 64);
    const s = '0x' + sig.slice(64, 128);
    const v = parseInt(sig.slice(128, 130), 16);
    
    // Execute transfer (this would use walletClient in practice)
    const hash = '0x...';
    return hash;
  }
  
  private generateNonce(): string {
    return keccak256(toBytes(Date.now().toString() + Math.random().toString()));
  }
  
  private async analyzeContract(address: string) {
    // Service implementation
    return {
      contractAddress: address,
      securityScore: 85,
      vulnerabilities: [],
      optimizations: ['Use custom errors instead of revert strings'],
      gasEstimate: 245000
    };
  }
  
  private async createDeliveryAttestation(
    client: string,
    contractAddress: string,
    result: any
  ): Promise<string> {
    // Create EAS attestation for service delivery
    return '0xAttestationUID...';
  }
  
  start(port: number = 3000) {
    this.app.listen(port, () => {
      console.log('Agent service provider listening on port', port);
    });
  }
}

const provider = new AgentServiceProvider();
provider.start();
```

## Gasless ERC-3009 USDC Transfers

ERC-3009 enables gasless transfers where a third party (facilitator) executes the transfer on behalf of the payer.

```typescript
import { createWalletClient, http, encodePacked, keccak256 } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

class USDCPaymentClient {
  private walletClient;
  private account;
  
  constructor(privateKey: string) {
    this.account = privateKeyToAccount(privateKey as `0x${string}`);
    
    this.walletClient = createWalletClient({
      account: this.account,
      chain: base,
      transport: http('https://mainnet.base.org')
    });
  }
  
  async createPaymentAuthorization(
    recipient: string,
    amount: string,
    validAfter: number,
    validBefore: number,
    nonce: string
  ): Promise<PaymentProof> {
    const domain = {
      name: 'USD Coin',
      version: '2',
      chainId: 8453,
      verifyingContract: USDC_BASE
    };
    
    const types = {
      TransferWithAuthorization: [
        { name: 'from', type: 'address' },
        { name: 'to', type: 'address' },
        { name: 'value', type: 'uint256' },
        { name: 'validAfter', type: 'uint256' },
        { name: 'validBefore', type: 'uint256' },
        { name: 'nonce', type: 'bytes32' }
      ]
    };
    
    const message = {
      from: this.account.address,
      to: recipient,
      value: BigInt(amount),
      validAfter: BigInt(validAfter),
      validBefore: BigInt(validBefore),
      nonce: nonce
    };
    
    const signature = await this.walletClient.signTypedData({
      domain,
      types,
      primaryType: 'TransferWithAuthorization',
      message
    });
    
    return {
      from: this.account.address,
      to: recipient,
      value: amount,
      validAfter: validAfter.toString(),
      validBefore: validBefore.toString(),
      nonce: nonce,
      signature: signature
    };
  }
  
  async payForService(serviceUrl: string, requestBody: any): Promise<any> {
    // Step 1: Request service without payment
    const initialResponse = await fetch(serviceUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(requestBody)
    });
    
    if (initialResponse.status !== 402) {
      return await initialResponse.json();
    }
    
    // Step 2: Receive payment challenge
    const challenge = await initialResponse.json();
    const payment = challenge.payment as PaymentChallenge;
    
    console.log('Payment required:', payment.amount, 'USDC');
    
    // Step 3: Create payment authorization
    const authorization = await this.createPaymentAuthorization(
      payment.recipient,
      payment.amount,
      payment.validAfter,
      payment.validBefore,
      payment.nonce
    );
    
    // Step 4: Submit payment proof
    const authToken = Buffer.from(JSON.stringify(authorization)).toString('base64');
    
    const paidResponse = await fetch(serviceUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + authToken
      },
      body: JSON.stringify(requestBody)
    });
    
    if (!paidResponse.ok) {
      throw new Error('Payment verification failed');
    }
    
    return await paidResponse.json();
  }
}

// Usage
const client = new USDCPaymentClient(process.env.PRIVATE_KEY!);

const result = await client.payForService(
  'https://agent.example.com/api/analyze-contract',
  { contractAddress: '0x742d35Cc6634C0532925a3b844Bc454e4438f44e' }
);

console.log('Service result:', result);
```

## Facilitator Architecture

Facilitators execute gasless transfers on behalf of payers, enabling seamless payment flows.

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

class PaymentFacilitator {
  private publicClient;
  private walletClient;
  private processedNonces: Set<string>;
  
  constructor(facilitatorPrivateKey: string) {
    const account = privateKeyToAccount(facilitatorPrivateKey as `0x${string}`);
    
    this.publicClient = createPublicClient({
      chain: base,
      transport: http('https://mainnet.base.org')
    });
    
    this.walletClient = createWalletClient({
      account,
      chain: base,
      transport: http('https://mainnet.base.org')
    });
    
    this.processedNonces = new Set();
  }
  
  async processPayment(proof: PaymentProof): Promise<{
    success: boolean;
    transactionHash?: string;
    error?: string;
  }> {
    // Validate payment proof
    if (this.processedNonces.has(proof.nonce)) {
      return { success: false, error: 'Nonce already processed' };
    }
    
    // Verify signature
    const isValid = await this.verifySignature(proof);
    if (!isValid) {
      return { success: false, error: 'Invalid signature' };
    }
    
    // Check validity window
    const now = Math.floor(Date.now() / 1000);
    const validAfter = parseInt(proof.validAfter);
    const validBefore = parseInt(proof.validBefore);
    
    if (now < validAfter || now > validBefore) {
      return { success: false, error: 'Payment authorization expired' };
    }
    
    try {
      // Execute transfer on-chain
      const txHash = await this.executeTransferOnChain(proof);
      
      // Mark nonce as processed
      this.processedNonces.add(proof.nonce);
      
      // Calculate and take facilitator fee (optional)
      await this.collectFee(proof, txHash);
      
      return {
        success: true,
        transactionHash: txHash
      };
    } catch (error) {
      return {
        success: false,
        error: 'Transaction execution failed'
      };
    }
  }
  
  private async verifySignature(proof: PaymentProof): Promise<boolean> {
    const domain = {
      name: 'USD Coin',
      version: '2',
      chainId: 8453,
      verifyingContract: USDC_BASE
    };
    
    const types = {
      TransferWithAuthorization: [
        { name: 'from', type: 'address' },
        { name: 'to', type: 'address' },
        { name: 'value', type: 'uint256' },
        { name: 'validAfter', type: 'uint256' },
        { name: 'validBefore', type: 'uint256' },
        { name: 'nonce', type: 'bytes32' }
      ]
    };
    
    const message = {
      from: proof.from,
      to: proof.to,
      value: BigInt(proof.value),
      validAfter: BigInt(proof.validAfter),
      validBefore: BigInt(proof.validBefore),
      nonce: proof.nonce
    };
    
    // Signature verification logic here
    return true;
  }
  
  private async executeTransferOnChain(proof: PaymentProof): Promise<string> {
    const usdcAbi = [{
      name: 'transferWithAuthorization',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'from', type: 'address' },
        { name: 'to', type: 'address' },
        { name: 'value', type: 'uint256' },
        { name: 'validAfter', type: 'uint256' },
        { name: 'validBefore', type: 'uint256' },
        { name: 'nonce', type: 'bytes32' },
        { name: 'v', type: 'uint8' },
        { name: 'r', type: 'bytes32' },
        { name: 's', type: 'bytes32' }
      ],
      outputs: []
    }];
    
    const sig = proof.signature.slice(2);
    const r = '0x' + sig.slice(0, 64);
    const s = '0x' + sig.slice(64, 128);
    const v = parseInt(sig.slice(128, 130), 16);
    
    const hash = await this.walletClient.writeContract({
      address: USDC_BASE,
      abi: usdcAbi,
      functionName: 'transferWithAuthorization',
      args: [
        proof.from as `0x${string}`,
        proof.to as `0x${string}`,
        BigInt(proof.value),
        BigInt(proof.validAfter),
        BigInt(proof.validBefore),
        proof.nonce as `0x${string}`,
        v,
        r as `0x${string}`,
        s as `0x${string}`
      ]
    });
    
    const receipt = await this.publicClient.waitForTransactionReceipt({ hash });
    return receipt.transactionHash;
  }
  
  private async collectFee(proof: PaymentProof, txHash: string): Promise<void> {
    // Optional: Implement fee collection logic
    // e.g., take 0.1% of payment amount
    const feeAmount = BigInt(proof.value) / BigInt(1000);
    console.log('Facilitator fee:', feeAmount.toString(), 'USDC');
  }
}

// Usage
const facilitator = new PaymentFacilitator(process.env.FACILITATOR_KEY!);

const payment: PaymentProof = {
  from: '0xPayer...',
  to: '0xServiceProvider...',
  value: '1000000',
  validAfter: '1234567890',
  validBefore: '1234571490',
  nonce: '0x...',
  signature: '0x...'
};

const result = await facilitator.processPayment(payment);
console.log('Payment processed:', result);
```

## Complete Commerce Flow Example

```typescript
class AgentCommerceFlow {
  async executeFullFlow() {
    console.log('=== Agent Commerce Flow ===\n');
    
    // 1. Discover agent
    console.log('Step 1: Discovering agents...');
    const discovery = new AgentDiscoveryService();
    const agents = await discovery.discoverAgents('smart-contract-audit', 85);
    const selectedAgent = agents[0];
    console.log('Selected agent:', selectedAgent.metadata.name);
    
    // 2. Verify attestations
    console.log('\nStep 2: Verifying attestations...');
    const verifiedAttestations = selectedAgent.attestations.filter(a => a.valid);
    console.log('Valid attestations:', verifiedAttestations.length);
    
    // 3. Negotiate payment
    console.log('\nStep 3: Negotiating payment...');
    const serviceEndpoint = selectedAgent.metadata.serviceEndpoint;
    const paymentClient = new USDCPaymentClient(process.env.PRIVATE_KEY!);
    
    // 4. Execute payment and receive service
    console.log('\nStep 4: Executing payment and service...');
    const result = await paymentClient.payForService(
      serviceEndpoint + '/analyze-contract',
      { contractAddress: '0x742d35Cc6634C0532925a3b844Bc454e4438f44e' }
    );
    
    console.log('\nService result:', result.analysis);
    console.log('Delivery attestation:', result.attestation);
    
    // 5. Leave feedback attestation
    console.log('\nStep 5: Creating feedback attestation...');
    const feedbackAttestation = await this.createFeedbackAttestation(
      selectedAgent.address,
      result.attestation,
      95 // Quality score
    );
    
    console.log('Feedback attestation created:', feedbackAttestation);
    console.log('\n=== Flow Complete ===');
  }
  
  private async createFeedbackAttestation(
    agentAddress: string,
    serviceAttestationUID: string,
    qualityScore: number
  ): Promise<string> {
    // Create EAS attestation for feedback
    return '0xFeedbackAttestationUID...';
  }
}

// Execute full flow
const flow = new AgentCommerceFlow();
await flow.executeFullFlow();
```

## Common Patterns

**Payment Retry Logic**: Implement exponential backoff for failed payment transfers.

**Attestation Chaining**: Reference previous attestations in new attestations to build reputation chains.

**Multi-Service Bundles**: Batch multiple service requests with single payment authorization.

## Gotchas

**Nonce Management**: Never reuse nonces. Generate cryptographically random nonces for each payment.

**Signature Malleability**: Always verify signature format and reject malleable signatures.

**Time Synchronization**: Ensure client and server clocks are synchronized for validAfter/validBefore checks.

**Gas Price Volatility**: Facilitators should monitor gas prices and implement dynamic fee adjustments.

**Attestation Privacy**: Be careful about what data is included in attestations. All attestation data is publicly readable on-chain.