# Payments with OnchainKit

Accept crypto payments for products or execute custom contract interactions.

## Checkout Component

The `<Checkout />` component provides a complete payment flow for product purchases.

### Basic Checkout

```tsx
import { Checkout, CheckoutButton } from '@coinbase/onchainkit/checkout';

function ProductCheckout() {
  return (
    <Checkout 
      productId="your-product-id" // From Coinbase Commerce
    >
      <CheckoutButton coinbaseBranded />
    </Checkout>
  );
}
```

### Custom Checkout with Charge

```tsx
import { 
  Checkout, 
  CheckoutButton,
  CheckoutStatus 
} from '@coinbase/onchainkit/checkout';

function CustomCheckout() {
  const chargeHandler = async () => {
    // Create charge via your backend
    const response = await fetch('/api/create-charge', {
      method: 'POST',
      body: JSON.stringify({
        amount: '10.00',
        currency: 'USD',
        description: 'Premium Subscription',
      }),
    });
    const { chargeId } = await response.json();
    return chargeId;
  };

  return (
    <Checkout chargeHandler={chargeHandler}>
      <CheckoutButton text="Pay $10" />
      <CheckoutStatus />
    </Checkout>
  );
}
```

## Transaction Component

For custom contract calls, use the `<Transaction />` component.

### Basic Transaction

```tsx
import { 
  Transaction, 
  TransactionButton,
  TransactionStatus,
  TransactionStatusLabel,
  TransactionStatusAction,
} from '@coinbase/onchainkit/transaction';

function MintNFT() {
  const contracts = [{
    address: '0x123...', // Contract address
    abi: nftAbi,
    functionName: 'mint',
    args: [1], // Mint 1 NFT
  }];

  return (
    <Transaction
      contracts={contracts}
      chainId={8453} // Base mainnet
      onSuccess={(response) => {
        console.log('Transaction successful:', response);
      }}
      onError={(error) => {
        console.error('Transaction failed:', error);
      }}
    >
      <TransactionButton />
      <TransactionStatus>
        <TransactionStatusLabel />
        <TransactionStatusAction />
      </TransactionStatus>
    </Transaction>
  );
}
```

### Multiple Contract Calls (Batch)

```tsx
function BatchTransaction() {
  const contracts = [
    {
      address: '0xTokenAddress...',
      abi: erc20Abi,
      functionName: 'approve',
      args: ['0xSpender...', BigInt(1000000)],
    },
    {
      address: '0xProtocol...',
      abi: protocolAbi,
      functionName: 'deposit',
      args: [BigInt(1000000)],
    },
  ];

  return (
    <Transaction contracts={contracts}>
      <TransactionButton text="Approve & Deposit" />
    </Transaction>
  );
}
```

## USDC Payment Flows

### Direct USDC Transfer

```tsx
import { parseUnits } from 'viem';

const USDC_ADDRESS = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913'; // Base USDC

function USDCPayment({ recipient, amount }: { recipient: string; amount: string }) {
  const contracts = [{
    address: USDC_ADDRESS,
    abi: [{
      name: 'transfer',
      type: 'function',
      inputs: [
        { name: 'to', type: 'address' },
        { name: 'amount', type: 'uint256' },
      ],
      outputs: [{ type: 'bool' }],
    }],
    functionName: 'transfer',
    args: [recipient, parseUnits(amount, 6)], // USDC has 6 decimals
  }];

  return (
    <Transaction contracts={contracts}>
      <TransactionButton text={`Pay ${amount} USDC`} />
    </Transaction>
  );
}
```

### USDC with Approval Check

```tsx
import { useReadContract } from 'wagmi';
import { parseUnits } from 'viem';

function USDCPaymentWithApproval({ 
  spender, 
  amount 
}: { 
  spender: string; 
  amount: string;
}) {
  const { address } = useAccount();
  const amountWei = parseUnits(amount, 6);

  const { data: allowance } = useReadContract({
    address: USDC_ADDRESS,
    abi: erc20Abi,
    functionName: 'allowance',
    args: [address!, spender],
  });

  const needsApproval = !allowance || allowance < amountWei;

  const contracts = needsApproval
    ? [
        {
          address: USDC_ADDRESS,
          abi: erc20Abi,
          functionName: 'approve',
          args: [spender, amountWei],
        },
        {
          address: spender,
          abi: protocolAbi,
          functionName: 'pay',
          args: [amountWei],
        },
      ]
    : [
        {
          address: spender,
          abi: protocolAbi,
          functionName: 'pay',
          args: [amountWei],
        },
      ];

  return (
    <Transaction contracts={contracts}>
      <TransactionButton 
        text={needsApproval ? 'Approve & Pay' : 'Pay'} 
      />
    </Transaction>
  );
}
```

## Handling Transaction States

### Transaction Lifecycle

```tsx
import { useTransactionContext } from '@coinbase/onchainkit/transaction';

function TransactionStateHandler() {
  return (
    <Transaction
      contracts={contracts}
      onStatus={(status) => {
        switch (status) {
          case 'init':
            console.log('Ready to submit');
            break;
          case 'pending':
            console.log('Waiting for wallet confirmation');
            break;
          case 'submitted':
            console.log('Transaction submitted to network');
            break;
          case 'success':
            console.log('Transaction confirmed!');
            break;
          case 'error':
            console.log('Transaction failed');
            break;
        }
      }}
    >
      <TransactionButton />
      <TransactionStatus />
    </Transaction>
  );
}
```

### Custom Status UI

```tsx
function CustomTransactionUI() {
  return (
    <Transaction contracts={contracts}>
      {({ status, transactionHash, error }) => (
        <div>
          {status === 'pending' && (
            <div className="flex items-center gap-2">
              <Spinner />
              <span>Confirm in wallet...</span>
            </div>
          )}
          {status === 'submitted' && (
            <a 
              href={`https://basescan.org/tx/${transactionHash}`}
              target="_blank"
            >
              View on BaseScan
            </a>
          )}
          {status === 'success' && (
            <div className="text-green-500">Success!</div>
          )}
          {status === 'error' && (
            <div className="text-red-500">{error?.message}</div>
          )}
          <TransactionButton />
        </div>
      )}
    </Transaction>
  );
}
```

## Error Handling

### Common Error Types

```tsx
function RobustTransaction() {
  const handleError = (error: any) => {
    if (error.code === 4001) {
      // User rejected transaction
      toast.error('Transaction cancelled');
    } else if (error.message?.includes('insufficient funds')) {
      toast.error('Insufficient balance for gas');
    } else if (error.message?.includes('execution reverted')) {
      // Contract error - parse reason
      const reason = error.message.match(/reason="([^"]+)"/)?.[1];
      toast.error(`Transaction failed: ${reason || 'Unknown error'}`);
    } else {
      toast.error('Transaction failed');
      console.error(error);
    }
  };

  return (
    <Transaction
      contracts={contracts}
      onError={handleError}
    >
      <TransactionButton />
    </Transaction>
  );
}
```

### Retry Logic

```tsx
function TransactionWithRetry() {
  const [retryCount, setRetryCount] = useState(0);

  return (
    <Transaction
      contracts={contracts}
      onError={(error) => {
        if (retryCount < 3 && error.message?.includes('network')) {
          setRetryCount(prev => prev + 1);
          // Component will re-render and user can try again
        }
      }}
    >
      <TransactionButton 
        text={retryCount > 0 ? 'Retry Transaction' : 'Submit'} 
      />
    </Transaction>
  );
}
```

## Next Steps

- [Troubleshooting](./05-troubleshooting.md) - Fix common issues
