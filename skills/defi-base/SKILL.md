---
name: defi-base
description: Interact with DeFi protocols on Base including Aerodrome, Moonwell, and Morpho
trigger_phrases:
  - "swap tokens on aerodrome"
  - "provide liquidity on base"
  - "lend on moonwell"
  - "borrow from morpho"
  - "supply collateral to lending protocol"
  - "check lending rates"
  - "execute defi transaction"
version: 1.0.0
---

# DeFi on Base

## Guidelines

This skill enables interaction with major DeFi protocols deployed on Base L2. Use this skill when the user wants to:

- Swap tokens using decentralized exchanges (Aerodrome)
- Provide liquidity to AMM pools
- Supply assets to lending protocols (Moonwell, Morpho)
- Borrow assets against collateral
- Check lending rates and pool statistics
- Execute complex DeFi strategies

Before executing any DeFi transaction:
1. Verify the user has sufficient token balance and gas
2. Check token approvals -- most DeFi interactions require approve() first
3. Calculate slippage tolerance and set appropriate minimums
4. Set reasonable deadlines for time-sensitive operations
5. Simulate transactions when possible to catch reverts
6. Warn users about price impact on large swaps

For lending protocols:
- Explain collateral factors and liquidation thresholds
- Check health factors before and after operations
- Warn about interest rate model changes under different utilization
- Verify oracle prices are recent and reasonable

For DEX operations:
- Always use volatile pools for uncorrelated assets
- Use stable pools only for like-kind assets (USDC/DAI)
- Account for fees in route calculations
- Consider splitting large orders across multiple pools

Security considerations:
- Never store private keys in code
- Use smart wallet infrastructure when available (see smart-wallet skill)
- Validate all user inputs (amounts, addresses, slippage)
- Set maximum slippage bounds (typically 0.5-2%)
- Use multicall for atomic multi-step operations
- Check contract pause states before execution

When combining DeFi operations:
- Batch related transactions using multicall
- Consider gas optimization (approval batching)
- Handle partial fills gracefully
- Implement proper error recovery

## Examples

**Example 1: Swap USDC for WETH**
User: "Swap 100 USDC for WETH on Aerodrome"

Action:
1. Check USDC balance and approval to Aerodrome Router
2. Get quote from Router for 100 USDC -> WETH swap
3. Calculate minimum output with 0.5% slippage
4. Execute swapExactTokensForTokens with deadline
5. Return transaction hash and output amount

**Example 2: Supply to Moonwell**
User: "Supply 1000 USDC to Moonwell"

Action:
1. Check USDC balance and approval to Moonwell USDC market
2. Call supply() on market contract with 1000 USDC
3. Calculate mToken balance received
4. Display new supply APY and total supplied
5. Return transaction hash and confirmation

**Example 3: Check Lending Rates**
User: "What are the current lending rates on Base?"

Action:
1. Query Moonwell USDC market for supply/borrow APY
2. Query major Morpho Blue markets for rates
3. Display comparison table with utilization rates
4. Highlight best opportunities
5. Warn about any unusual rate spikes

## Resources

- Aerodrome Finance: https://aerodrome.finance/
- Aerodrome Docs: https://docs.aerodrome.finance/
- Moonwell Protocol: https://moonwell.fi/
- Moonwell Docs: https://docs.moonwell.fi/
- Morpho Blue: https://morpho.org/
- Morpho Docs: https://docs.morpho.org/
- Base DeFi Llama: https://defillama.com/chain/Base
- Base Contract Addresses: https://docs.base.org/contracts/
- Aerodrome Router: 0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43
- Moonwell USDC Market: 0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22
- USDC on Base: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

## Cross-References

- **smart-wallet**: Use smart wallet infrastructure for gasless DeFi transactions and session keys
- **viem-wagmi**: Core Web3 libraries for contract interaction and transaction building
- **solidity**: Understanding contract interfaces and DeFi protocol architecture
- **cctp**: Bridge assets to Base from other chains before DeFi operations
- **erc20-ops**: Token approvals, transfers, and balance checks required for DeFi
- **block-explorer**: Verify transaction status and debug failed DeFi operations