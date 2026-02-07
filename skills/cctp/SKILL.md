---
name: cctp
description: Cross-Chain Transfer Protocol (CCTP) for native USDC bridging between Base and other chains
trigger_phrases:
  - "bridge USDC to"
  - "transfer USDC cross-chain"
  - "send USDC to another chain"
  - "CCTP transfer"
  - "native USDC bridge"
  - "cross-chain USDC"
version: 1.0.0
---

# Circle Cross-Chain Transfer Protocol (CCTP)

## Guidelines

Use this skill when you need to:

1. **Bridge Native USDC**: Transfer USDC between Base and other supported chains (Ethereum, Arbitrum, Polygon, Optimism, Avalanche) while maintaining native USDC status -- not wrapped tokens.

2. **Minimize Bridge Risk**: CCTP uses a burn-and-mint model operated by Circle, eliminating third-party bridge risks and locked liquidity concerns that affect traditional bridges.

3. **Optimize Cross-Chain Operations**: When building applications that require USDC on multiple chains, CCTP ensures capital efficiency by minting native USDC directly rather than wrapping tokens.

4. **Programmatic Cross-Chain Transfers**: Implement automated cross-chain USDC flows in smart contracts or autonomous agents with full on-chain verification.

Do NOT use this skill for:
- Bridging non-USDC tokens (use standard bridges)
- Same-chain USDC transfers (use standard ERC-20 transfers)
- Bridging to unsupported chains (check supported chains first)
- Transfers requiring instant finality (CCTP requires attestation time)

**Key Concepts**:
- **Burn-and-Mint Model**: Source chain burns USDC, destination chain mints equivalent amount
- **Attestation Service**: Circle's attestation service validates burn events before minting
- **Native USDC**: Resulting tokens are native USDC on destination chain, not wrapped
- **Domain IDs**: Each chain has a unique domain identifier for routing
- **No Liquidity Pools**: Unlike traditional bridges, no locked liquidity required

**Integration Points**:
- TokenMessenger contract on Base: 0x1682Ae6375C4E4A97e4B583BC394c861A46D8962
- USDC token on Base: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
- Circle Attestation API: https://iris-api.circle.com/attestations/{messageHash}
- Destination chain TokenMessenger for receiving transfers

**Typical Flow**:
1. User approves USDC spend on source chain TokenMessenger
2. Call depositForBurn() on source TokenMessenger with amount and destination
3. Monitor burn event and retrieve message bytes
4. Poll Circle Attestation API for attestation signature
5. Call receiveMessage() on destination TokenMessenger with message and attestation
6. Native USDC minted to recipient on destination chain

**Timing Considerations**:
- Attestation typically available within 10-20 minutes
- Ethereum finality affects attestation timing for some routes
- Always implement proper timeout and retry logic
- Monitor transaction status on both chains

## Examples

**Example 1: Bridge USDC from Base to Ethereum**

Trigger: "Bridge 100 USDC from Base to Ethereum for address 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb1"

Action:
1. Check USDC balance on Base
2. Approve TokenMessenger to spend USDC
3. Call depositForBurn with amount, Ethereum domain ID (0), and recipient
4. Monitor for MessageSent event
5. Retrieve attestation from Circle API
6. Provide user with attestation and instructions for claiming on Ethereum

**Example 2: Automated Cross-Chain Yield Strategy**

Trigger: "Set up automated USDC transfer to Arbitrum when Base yield drops below 5%"

Action:
1. Monitor yield rates on Base using Moonwell/Aerodrome integrations
2. When threshold triggered, initiate CCTP transfer to Arbitrum
3. Store message hash and implement attestation polling
4. Auto-complete transfer on Arbitrum when attestation available
5. Deploy USDC into Arbitrum yield opportunities

**Example 3: Cross-Chain Payment Settlement**

Trigger: "Pay invoice of 500 USDC to Polygon address 0x1234... using CCTP"

Action:
1. Verify recipient address and Polygon as destination
2. Calculate total cost including source chain gas
3. Execute depositForBurn to Polygon domain (7)
4. Store transaction receipt and message details
5. Monitor attestation status and notify recipient
6. Optionally auto-complete on destination if agent has Polygon presence

## Resources

- **Circle CCTP Documentation**: https://developers.circle.com/stablecoins/docs/cctp-getting-started
- **CCTP Protocol Spec**: https://github.com/circlefin/evm-cctp-contracts
- **Base TokenMessenger**: https://basescan.org/address/0x1682Ae6375C4E4A97e4B583BC394c861A46D8962
- **Attestation API**: https://developers.circle.com/stablecoins/docs/cctp-attestation-service
- **Supported Chains**: https://developers.circle.com/stablecoins/docs/supported-domains
- **USDC on Base**: https://basescan.org/token/0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

## Cross-References

- **smart-wallet**: Use smart wallets for automated cross-chain strategies and gasless CCTP transfers
- **defi-base**: Combine with DeFi protocols for cross-chain yield optimization strategies
- **agent-protocol**: Implement autonomous cross-chain USDC management and rebalancing
- **attestation**: Leverage EAS for recording cross-chain transfer history and settlement proofs