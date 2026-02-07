# Client Implementation (The Payer)

To build an agent that can pay x402 gates, you must handle the 402 status code and sign EIP-712 messages.

## Core Logic

1.  **Interceptor:** Wrap your fetch or HTTP client. If response status is 402, pause.
2.  **Decode:** Parse the PAYMENT-REQUIRED header from the response.
3.  **Select Scheme:** Choose a supported scheme from the accepts array (usually exact or range).
4.  **Construct Payload:** Create a JSON object matching the requirement:
    {
      "scheme": "exact",
      "network": "base",
      "amount": "100000",
      "asset": "0x...",
      "payTo": "0x...",
      "nonce": "random-uuid"
    }
5.  **Sign (EIP-712):** Sign this payload using the Agent's wallet. The domain separator must match the x402 Protocol V1.
6.  **Retry:** Send the original request again with the PAYMENT-SIGNATURE header containing your signed payload.

## EIP-712 Domain Separator

const domain = {
  name: "x402",
  version: "1",
  chainId: 8453, // Base
  verifyingContract: "0x..." // Facilitator Address
};