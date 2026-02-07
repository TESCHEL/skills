---
name: x402-protocol
description: Implementation guide for the x402 open payment standard (HTTP 402).
trigger_phrases:
  - "implement x402"
  - "handle 402 payment required"
  - "create a paid API endpoint"
  - "pay for a resource autonomously"
  - "integrate coinbase payments"
version: 1.0.0
---

# x402 Protocol Specialist

This skill enables the agent to understand and implement the x402 standard, which allows for stateless, internet-native payments via HTTP headers. It covers both the Client (Payer) and Server (Earner) roles.

## Guidelines
1.  **Header Precision:** Always use the standard headers: PAYMENT-REQUIRED (Server->Client), PAYMENT-SIGNATURE (Client->Server), and PAYMENT-RESPONSE (Server->Client).
2.  **Statelessness:** x402 is stateless. Do not design flows that rely on cookies or sessions. The payment proof is in the request header.
3.  **Token Compatibility:** Verify that the payment asset implements **EIP-3009** (transferWithAuthorization). Standard USDC on Base is the preferred asset.
4.  **Facilitators:** For server implementations, default to using the Coinbase Facilitator (https://www.x402.org/facilitator) for verification unless running a custom node.

## Examples

**User/Trigger:** "How do I make an API endpoint that charges 1 USDC?"
**Agent Action:** Consult 03-server-implementation.md.
**Good Output:** "You need to use the x402-express middleware. Configure the 'exact' scheme with an amount of 1000000 (USDC has 6 decimals) and the Base chain ID."

**User/Trigger:** "My agent received a 402 error. What now?"
**Agent Action:** Consult 02-client-implementation.md.
**Good Output:** "The agent must parse the PAYMENT-REQUIRED header to get the payTo address and amount, sign an EIP-712 payload, and retry the request with the PAYMENT-SIGNATURE header."

## Resources
* [Protocol Spec](instructions/01-protocol-spec.md): The core HTTP handshake flow.
* [Client Implementation](instructions/02-client-implementation.md): How to pay.
* [Server Implementation](instructions/03-server-implementation.md): How to charge.
* [Smart Contracts](instructions/04-smart-contracts.md): EIP-3009 and Facilitator details.