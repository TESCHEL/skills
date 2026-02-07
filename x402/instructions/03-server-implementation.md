# Server Implementation (The Earner)

To monetize an API with x402, use the middleware pattern. You do not need to run your own blockchain node; you can delegate verification to a Facilitator.

## Middleware Setup (Node/Express)

import { paymentMiddleware } from '@x402/express';

app.use('/premium-endpoint', paymentMiddleware({
  store: new MemoryStore(), // Or Redis for production
  schemes: [{
    scheme: 'exact',
    config: {
      network: 'base',
      amount: BigInt(100000), // 0.10 USDC
      asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
      payTo: process.env.SERVER_WALLET_ADDRESS
    }
  }],
  facilitator: 'https://www.x402.org/facilitator'
}));

## How It Works
1. **Interception:** The middleware catches requests to /premium-endpoint.
2. **Check:** If PAYMENT-SIGNATURE is missing, it returns 402 with your configured price.
3. **Verify:** If header exists, it posts the payload to the Facilitator URL.
4. **Settle:** The Facilitator checks the signature and broadcasts the transaction to Base.
5. **Serve:** Once confirmed, the middleware calls next() and your API logic runs.