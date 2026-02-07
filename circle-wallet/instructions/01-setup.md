# Wallet Types

Circle provides two distinct wallet architectures for Base blockchain integration: User-Controlled Wallets and Developer-Controlled Wallets. Each approach offers different trade-offs in key custody, signing mechanisms, and operational control.

## User-Controlled Wallets

User-Controlled Wallets place cryptographic key management directly on the end user's device. The user maintains full custody of their private keys, which never leave their mobile or web environment. Circle's SDK facilitates secure key generation, storage, and transaction signing locally.

### Architecture Overview

In the User-Controlled model:

- Private keys are generated on the user's device using secure hardware enclaves (iOS Secure Enclave, Android Keystore)
- Keys are protected by PIN codes or biometric authentication (Face ID, Touch ID, fingerprint)
- Circle's SDK handles challenge-response flows with Circle's API for transaction validation
- The application never has direct access to private keys
- Users can recover wallets through secure backup mechanisms

### SDK Integration

User-Controlled Wallets leverage the `@circle-fin/user-controlled-wallets` SDK for JavaScript/TypeScript applications:

```javascript
import { CircleWalletSdk } from '@circle-fin/user-controlled-wallets';

// Initialize SDK with your Circle App ID
const sdk = new CircleWalletSdk();
await sdk.init({
  appId: 'your-app-id-from-circle-console',
  endpoint: 'https://api.circle.com/v1/w3s'
});

// Create a new wallet (keys generated on user's device)
const challenge = await fetch('https://api.circle.com/v1/w3s/user/initialize', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY'
  },
  body: JSON.stringify({
    blockchains: ['BASE']
  })
});

const challengeData = await challenge.json();

// SDK executes challenge locally with user's PIN/biometric
const result = await sdk.execute(challengeData.challengeId, (error, result) => {
  if (error) {
    console.error('Wallet creation failed:', error);
    return;
  }
  console.log('Wallet created:', result.walletAddress);
  // Wallet address example: 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb1
});
```

### Challenge-Response Flow

Circle's User-Controlled Wallet security model uses challenge-response authentication:

1. Application requests an operation (create wallet, sign transaction) from Circle API
2. Circle returns a challenge ID and encrypted challenge data
3. SDK decrypts challenge on-device using local keys
4. User authorizes with PIN or biometric
5. SDK signs response and sends back to Circle
6. Circle validates signature and executes operation

This architecture ensures private keys never transit the network.

### Transaction Signing

Signing transactions with User-Controlled Wallets:

```javascript
// Request transaction challenge from your backend
const txChallenge = await fetch('https://api.circle.com/v1/w3s/user/transactions/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY',
    'X-User-Token': userToken
  },
  body: JSON.stringify({
    userId: 'user-123',
    walletId: 'wallet-456',
    blockchain: 'BASE',
    tokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
    destinationAddress: '0x8B7B3b6F0C2B5e2E9F5A3C1D7E9B4A6C8D2F1E3A',
    amount: '10.50'
  })
});

const txChallengeData = await txChallenge.json();

// Execute signing on user's device
await sdk.execute(txChallengeData.challengeId, (error, result) => {
  if (error) {
    console.error('Transaction signing failed:', error);
    return;
  }
  console.log('Transaction submitted:', result.transactionHash);
});
```

### Mobile Integration

For native mobile applications, Circle provides native SDKs:

**iOS (Swift):**

```swift
import CircleWalletSdk

let sdk = WalletSdk.shared
sdk.initialize(appId: "your-app-id-from-circle-console")

// Execute challenge with biometric prompt
sdk.execute(challengeId: challengeId) { result in
    switch result {
    case .success(let data):
        print("Wallet address: \(data.walletAddress)")
    case .failure(let error):
        print("Error: \(error.localizedDescription)")
    }
}
```

**Android (Kotlin):**

```kotlin
import com.circle.wallet.sdk.WalletSdk

val sdk = WalletSdk.getInstance()
sdk.initialize(applicationContext, "your-app-id-from-circle-console")

sdk.execute(challengeId) { result ->
    result.onSuccess { data ->
        println("Wallet address: ${data.walletAddress}")
    }.onFailure { error ->
        println("Error: ${error.message}")
    }
}
```

## Developer-Controlled Wallets

Developer-Controlled Wallets shift key custody to Circle's infrastructure. Circle manages private keys using Multi-Party Computation (MPC) and Hardware Security Modules (HSM). Your application initiates transactions server-side through Circle's REST API.

### Architecture Overview

In the Developer-Controlled model:

- Private keys are generated and stored in Circle's secure infrastructure
- MPC technology distributes key material across multiple parties
- HSMs provide hardware-level security for cryptographic operations
- Your server makes API calls to trigger wallet operations
- No client-side key management required
- Circle handles gas fee sponsorship and transaction broadcasting

### Creating a Developer-Controlled Wallet

Developer-Controlled Wallets require server-side API integration. First, create a wallet set to organize your wallets:

```javascript
// Node.js example with fetch
const createWalletSet = async () => {
  const response = await fetch('https://api.circle.com/v1/w3s/developer/walletSets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer YOUR_API_KEY'
    },
    body: JSON.stringify({
      name: 'Base Production Wallets'
    })
  });
  
  const data = await response.json();
  return data.data.walletSet.id; // e.g., "01928374-5678-abcd-ef01-234567890abc"
};

const walletSetId = await createWalletSet();
```

Now create a wallet within that set:

```javascript
const createWallet = async (walletSetId, entitySecretCiphertext) => {
  const response = await fetch('https://api.circle.com/v1/w3s/developer/wallets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer YOUR_API_KEY',
      'X-Entity-Secret': entitySecretCiphertext
    },
    body: JSON.stringify({
      blockchains: ['BASE'],
      count: 1,
      walletSetId: walletSetId,
      metadata: [
        {
          name: 'User Wallet',
          refId: 'user-12345'
        }
      ]
    })
  });
  
  const data = await response.json();
  const wallet = data.data.wallets[0];
  
  console.log('Wallet ID:', wallet.id);
  console.log('Base Address:', wallet.address);
  console.log('State:', wallet.state); // LIVE
  
  return wallet;
};

// Example output:
// Wallet ID: 019a1234-5678-9abc-def0-123456789abc
// Base Address: 0x9A8B7C6D5E4F3A2B1C0D9E8F7A6B5C4D3E2F1A0B
// State: LIVE
```

**Python Example:**

```python
import requests
import json

def create_wallet_set():
    url = "https://api.circle.com/v1/w3s/developer/walletSets"
    headers = {
        "Content-Type": "application/json",
        "Authorization": "Bearer YOUR_API_KEY"
    }
    payload = {"name": "Base Production Wallets"}
    
    response = requests.post(url, headers=headers, json=payload)
    data = response.json()
    return data["data"]["walletSet"]["id"]

def create_developer_wallet(wallet_set_id, entity_secret):
    url = "https://api.circle.com/v1/w3s/developer/wallets"
    headers = {
        "Content-Type": "application/json",
        "Authorization": "Bearer YOUR_API_KEY",
        "X-Entity-Secret": entity_secret
    }
    payload = {
        "blockchains": ["BASE"],
        "count": 1,
        "walletSetId": wallet_set_id,
        "metadata": [{"name": "User Wallet", "refId": "user-12345"}]
    }
    
    response = requests.post(url, headers=headers, json=payload)
    data = response.json()
    wallet = data["data"]["wallets"][0]
    
    print(f"Wallet ID: {wallet['id']}")
    print(f"Base Address: {wallet['address']}")
    print(f"State: {wallet['state']}")
    
    return wallet

wallet_set_id = create_wallet_set()
wallet = create_developer_wallet(wallet_set_id, "your-entity-secret-ciphertext")
```

### Token Transfers

Initiate token transfers from Developer-Controlled Wallets:

```javascript
const transferUSDC = async (walletId, destinationAddress, amount, entitySecret) => {
  const response = await fetch('https://api.circle.com/v1/w3s/developer/transactions/transfer', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer YOUR_API_KEY',
      'X-Entity-Secret': entitySecret
    },
    body: JSON.stringify({
      walletId: walletId,
      blockchain: 'BASE',
      tokenAddress: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
      destinationAddress: destinationAddress,
      amount: amount,
      fee: {
        type: 'level',
        config: {
          feeLevel: 'MEDIUM' // LOW, MEDIUM, HIGH
        }
      }
    })
  });
  
  const data = await response.json();
  console.log('Transaction ID:', data.data.id);
  console.log('State:', data.data.state); // INITIATED -> SENT -> CONFIRMED
  
  return data.data;
};

// Transfer 25.00 USDC
await transferUSDC(
  '019a1234-5678-9abc-def0-123456789abc',
  '0x8B7B3b6F0C2B5e2E9F5A3C1D7E9B4A6C8D2F1E3A',
  '25.00',
  'your-entity-secret-ciphertext'
);
```

### Checking Balances

Retrieve token balances for Developer-Controlled Wallets:

```javascript
const getWalletBalances = async (walletId) => {
  const response = await fetch(
    `https://api.circle.com/v1/w3s/wallets/${walletId}/balances`,
    {
      headers: {
        'Authorization': 'Bearer YOUR_API_KEY'
      }
    }
  );
  
  const data = await response.json();
  
  data.data.tokenBalances.forEach(balance => {
    console.log(`Token: ${balance.token.symbol}`);
    console.log(`Amount: ${balance.amount}`);
    console.log(`Address: ${balance.token.tokenAddress}`);
  });
  
  return data.data.tokenBalances;
};

// Example output:
// Token: USDC
// Amount: 1250.500000
// Address: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
```

**Python Example:**

```python
def get_wallet_balances(wallet_id, api_key):
    url = f"https://api.circle.com/v1/w3s/wallets/{wallet_id}/balances"
    headers = {"Authorization": f"Bearer {api_key}"}
    
    response = requests.get(url, headers=headers)
    data = response.json()
    
    for balance in data["data"]["tokenBalances"]:
        print(f"Token: {balance['token']['symbol']}")
        print(f"Amount: {balance['amount']}")
        print(f"Address: {balance['token']['tokenAddress']}")
    
    return data["data"]["tokenBalances"]
```

### Entity Secret

The Entity Secret is a critical security component for Developer-Controlled Wallets. It's an encrypted key that authorizes wallet operations:

- Generated during initial Circle account setup
- Used as `X-Entity-Secret` header in API requests
- Encrypted using your Circle API key
- Must be stored securely (environment variables, secret management system)
- Never expose in client-side code or version control

Circle provides an Entity Secret ciphertext in your developer console. Store it as:

```bash
CIRCLE_ENTITY_SECRET=your-encrypted-entity-secret-ciphertext-here
```

## Comparison: User-Controlled vs Developer-Controlled

| Aspect | User-Controlled Wallets | Developer-Controlled Wallets |
|--------|------------------------|-----------------------------|
| **Key Custody** | User's device (Secure Enclave/Keystore) | Circle's MPC/HSM infrastructure |
| **Private Key Access** | Never leaves user's device | Managed by Circle, inaccessible to developer |
| **Signing Location** | Client-side (mobile/web) | Server-side via Circle API |
| **User Experience** | Requires PIN/biometric for each transaction | Seamless, no user prompts |
| **Recovery** | User-managed backup phrases or social recovery | Circle-managed, recoverable via API |
| **Security Model** | User responsibility, non-custodial | Circle responsibility, custodial |
| **Gas Handling** | User must hold native tokens (ETH on Base) | Circle sponsors gas fees |
| **Implementation** | SDK integration in app | REST API integration on server |
| **Best For** | Consumer wallets, DeFi, self-custody | Custodial services, enterprise, automated operations |
| **Regulatory** | Non-custodial, less regulatory burden | Custodial, may require licenses |
| **Scalability** | Limited by client device performance | High throughput server-side operations |
| **Transaction Speed** | Dependent on user interaction | Automated, faster execution |

## Wallet Sets

Wallet Sets provide organizational structure for Developer-Controlled Wallets:

- Group related wallets together (e.g., by environment, purpose, or customer)
- Apply consistent policies across wallet groups
- Simplify wallet management and reporting
- Enable batch operations

```javascript
// Create wallet sets for different purposes
const productionSet = await createWalletSet('Production Wallets');
const testingSet = await createWalletSet('Testing Wallets');
const escrowSet = await createWalletSet('Escrow Wallets');

// List all wallet sets
const listWalletSets = async () => {
  const response = await fetch('https://api.circle.com/v1/w3s/developer/walletSets', {
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY'
    }
  });
  return await response.json();
};
```

## Integration with Smart Wallets and X402

Circle wallets integrate seamlessly with Base's smart wallet infrastructure and X402 payment protocol:

**Smart Wallet Integration:**

Both User-Controlled and Developer-Controlled wallets can interact with smart contract wallets. See the `smart-wallet` skill for:

- Deploying ERC-4337 account abstraction wallets
- Batch transaction capabilities
- Session key management
- Gas sponsorship patterns

User-Controlled wallets can sign UserOperations for smart wallets, while Developer-Controlled wallets can programmatically manage smart wallet deployments.

**X402 Payment Protocol:**

Developer-Controlled wallets are ideal for X402 payment endpoints. See the `x402` skill for:

- HTTP 402 Payment Required responses
- Circle wallet-based payment verification
- USDC payment flows on Base
- Automated payment processing

Example flow: Your API returns 402 with a Circle wallet payment address -> client sends USDC from their Circle wallet -> your backend verifies payment via Circle API -> grant access.

## Security Considerations

**User-Controlled Wallets:**

- Implement rate limiting on challenge requests
- Validate challenge responses server-side
- Use secure communication channels (HTTPS, certificate pinning)
- Implement app attestation (iOS DeviceCheck, Android SafetyNet)
- Never log or transmit sensitive key material

**Developer-Controlled Wallets:**

- Rotate API keys regularly
- Use separate API keys for production and testing
- Implement IP whitelisting for API access
- Store Entity Secret in secure vault (AWS Secrets Manager, HashiCorp Vault)
- Monitor wallet activity for anomalies
- Implement withdrawal limits and approval workflows
- Use webhook notifications for transaction monitoring

## Choosing the Right Wallet Type

Select User-Controlled Wallets when:

- Building consumer-facing applications requiring self-custody
- Users need full control over their assets
- Compliance requires non-custodial architecture
- Integrating with DeFi protocols where user signatures are required
- Mobile-first experiences with biometric authentication

Select Developer-Controlled Wallets when:

- Building custodial services or managed wallets
- Automating treasury operations or payment flows
- Creating seamless user experiences without signature prompts
- Implementing high-frequency trading or automated strategies
- Enterprise applications requiring centralized control
- Simplifying onboarding with no wallet setup required

## Next Steps

After choosing your wallet type:

1. Set up Circle developer account at https://console.circle.com
2. Generate API keys and Entity Secret
3. Configure wallet settings and blockchain networks
4. Implement wallet creation and transaction flows
5. Test thoroughly on Base testnet before production
6. Monitor wallet activity through Circle's dashboard
7. Integrate with smart wallets (see `smart-wallet` skill)
8. Implement X402 payment flows (see `x402` skill)

Both wallet types provide robust foundations for Base blockchain applications, with Circle handling infrastructure complexity while you focus on building great user experiences.