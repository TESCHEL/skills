# Smart Contract & Asset Requirements

x402 relies on specific smart contract standards to enable gasless, one-step payments.

## EIP-3009: Transfer With Authorization
The payment asset MUST support EIP-3009. This allows the user to sign a message ("I authorize paying 1 USDC to Server") which the Server/Facilitator then submits to the blockchain. This means the Server pays the gas, not the Client.

### Supported Assets (Base)
* **USDC:** 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 (Native USDC)
* Note: Standard ERC-20 tokens are NOT supported unless wrapped in an EIP-3009 adapter.

## Facilitators
A Facilitator is a trusted relayer that accepts the signed message and submits the transaction.
* **Coinbase Facilitator (Base):** https://www.x402.org/facilitator
* **Facilitator Contract (Base):** 0x... (Check https://docs.x402.org for latest deployments)

## Gas & Economics
* **Payer (Client):** Pays 0 gas. Pays only the asset amount.
* **Facilitator/Server:** Pays the gas fee to submit the transaction.
* **Economics:** Ensure your API price covers the gas cost of settlement (~$0.01 on Base), or batch payments using the deferred scheme.