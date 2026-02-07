# Cast Command-Line Tools

## Overview

Cast is Foundry's command-line tool for performing Ethereum RPC calls, sending transactions, encoding data, managing wallets, and interacting with contracts. This lesson covers essential cast commands for Base blockchain interaction.

## Reading Contract State

Query contract state without gas costs:

```bash
# Call a contract view function
cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "balanceOf(address)(uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org

# Get ERC20 token symbol
cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "symbol()(string)" \
  --rpc-url https://mainnet.base.org

# Get token decimals
cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "decimals()(uint8)" \
  --rpc-url https://mainnet.base.org

# Call with multiple return values
cast call 0x4200000000000000000000000000000000000021 \
  "getAttestation(bytes32)((bytes32,uint64,uint64,uint64,address,address,bool,bytes,bytes))" \
  0x1234... \
  --rpc-url https://mainnet.base.org

# Call at specific block
cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "totalSupply()(uint256)" \
  --rpc-url https://mainnet.base.org \
  --block 10000000
```

## Sending Transactions

Send state-changing transactions:

```bash
# Send ETH
cast send 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --value 0.1ether \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY

# Call contract function (ERC20 transfer)
cast send 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "transfer(address,uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  1000000 \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY

# Send with specific gas limit
cast send 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "approve(address,uint256)" \
  0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43 \
  1000000000 \
  --gas-limit 100000 \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY

# Send with confirmation wait
cast send 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "transfer(address,uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  1000000 \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY \
  --confirmations 3

# Send with legacy transaction (no EIP-1559)
cast send 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --value 0.1ether \
  --gas-price 1000000000 \
  --legacy \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY
```

## Gas Estimation

Estimate gas for transactions:

```bash
# Estimate gas for function call
cast estimate 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "transfer(address,uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  1000000 \
  --rpc-url https://mainnet.base.org

# Estimate with specific caller
cast estimate 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "transfer(address,uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  1000000 \
  --from 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org

# Get current gas price
cast gas-price --rpc-url https://mainnet.base.org

# Get current base fee
cast basefee --rpc-url https://mainnet.base.org

# Estimate deployment gas
cast estimate --create $(cat out/MyToken.sol/MyToken.json | jq -r .bytecode.object) \
  --rpc-url https://mainnet.base.org
```

## ABI Encoding and Decoding

Encode and decode function calls and data:

```bash
# Encode function call
cast calldata "transfer(address,uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  1000000

# Output: 0xa9059cbb000000000000000000000000742d35cc6634c0532925a3b844bc9e7595f0beb00000000000000000000000000000000000000000000000000000000000f4240

# Encode constructor arguments
cast abi-encode "constructor(string,string,uint256)" \
  "MyToken" \
  "MTK" \
  1000000000000000000000000

# Decode function call
cast --calldata-decode "transfer(address,uint256)" \
  0xa9059cbb000000000000000000000000742d35cc6634c0532925a3b844bc9e7595f0beb00000000000000000000000000000000000000000000000000000000000f4240

# Decode return data
cast --abi-decode "balanceOf(address)(uint256)" \
  0x00000000000000000000000000000000000000000000000000000000000f4240

# Get function signature (selector)
cast sig "transfer(address,uint256)"
# Output: 0xa9059cbb

# Get event signature
cast sig-event "Transfer(address,address,uint256)"
# Output: 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef

# Compute function selector from signature
cast keccak "transfer(address,uint256)" | cast --to-bytes32 | cut -c1-10
```

## Blockchain Data Queries

Query blockchain state:

```bash
# Get latest block number
cast block-number --rpc-url https://mainnet.base.org

# Get block details
cast block latest --rpc-url https://mainnet.base.org
cast block 10000000 --rpc-url https://mainnet.base.org

# Get transaction details
cast tx 0x742d35cc6634c0532925a3b844bc9e7595f0beb000000000000000000000001 \
  --rpc-url https://mainnet.base.org

# Get transaction receipt
cast receipt 0x742d35cc6634c0532925a3b844bc9e7595f0beb000000000000000000000001 \
  --rpc-url https://mainnet.base.org

# Get account balance
cast balance 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org

# Get account nonce
cast nonce 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org

# Get contract code
cast code 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  --rpc-url https://mainnet.base.org

# Get storage slot value
cast storage 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  0 \
  --rpc-url https://mainnet.base.org

# Get chain ID
cast chain-id --rpc-url https://mainnet.base.org
# Output: 8453

# Get chain info
cast chain --rpc-url https://mainnet.base.org
```

## Log and Event Queries

Query contract events:

```bash
# Get logs for Transfer events
cast logs \
  --from-block 10000000 \
  --to-block 10001000 \
  --address 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "Transfer(address indexed from, address indexed to, uint256 value)" \
  --rpc-url https://mainnet.base.org

# Get logs with specific indexed parameters
cast logs \
  --from-block 10000000 \
  --to-block 10001000 \
  --address 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "Transfer(address indexed from, address indexed to, uint256 value)" \
  --from 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org

# Get recent logs
cast logs \
  --from-block -1000 \
  --address 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "Transfer(address indexed from, address indexed to, uint256 value)" \
  --rpc-url https://mainnet.base.org

# Get logs and decode
cast logs \
  --from-block 10000000 \
  --to-block 10001000 \
  --address 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "Transfer(address indexed from, address indexed to, uint256 value)" \
  --json \
  --rpc-url https://mainnet.base.org | jq
```

## Wallet Management

Manage wallets and keys:

```bash
# Generate new wallet
cast wallet new

# Generate with specific mnemonic length
cast wallet new --mnemonic-length 24

# Derive address from private key
cast wallet address --private-key $PRIVATE_KEY

# Derive address from mnemonic
cast wallet address --mnemonic "test test test test test test test test test test test junk"

# Sign message
cast wallet sign "Hello Base" --private-key $PRIVATE_KEY

# Verify signature
cast wallet verify \
  --address 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  "Hello Base" \
  0x1234567890abcdef...

# Get vanity address
cast wallet vanity --starts-with dead

# Convert between address formats
cast --to-checksum-address 0x833589fcd6edb6e08f4c7c32d4f71b54bda02913
# Output: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
```

## Data Conversion

Convert between formats:

```bash
# Convert hex to decimal
cast --to-dec 0xFF
# Output: 255

# Convert decimal to hex
cast --to-hex 255
# Output: 0xff

# Convert to wei
cast --to-wei 1 ether
# Output: 1000000000000000000

# Convert from wei
cast --from-wei 1000000000000000000
# Output: 1.000000000000000000

# Convert to fixed bytes
cast --to-bytes32 0x1234
# Output: 0x1234000000000000000000000000000000000000000000000000000000000000

# Compute keccak256 hash
cast keccak "Hello Base"
# Output: 0x...

# Format units (e.g., USDC with 6 decimals)
cast --to-unit 1000000 6
# Output: 1.000000

# Parse units
cast --from-unit 1.5 6
# Output: 1500000
```

## ENS and Name Resolution

Resolve names on Base (Basenames):

```bash
# Resolve basename to address
cast resolve-name alice.base.eth --rpc-url https://mainnet.base.org

# Reverse resolve address to basename
cast lookup-address 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org

# Get basename owner
cast call 0xb94704422c2a1e396835a571837aa5ae53285a95 \
  "owner(bytes32)(address)" \
  $(cast --to-bytes32 $(cast keccak "alice.base.eth")) \
  --rpc-url https://mainnet.base.org
```

## Useful One-Liners for Base

Practical command combinations:

```bash
# Check USDC balance in human-readable format
cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "balanceOf(address)(uint256)" \
  0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
  --rpc-url https://mainnet.base.org | \
  xargs -I {} cast --to-unit {} 6

# Monitor USDC transfers to address
while true; do 
  cast logs --from-block latest \
    --address 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
    "Transfer(address indexed, address indexed to, uint256)" \
    --to 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb \
    --rpc-url https://mainnet.base.org
  sleep 2
done

# Get contract deployment block
cast block-number --rpc-url https://mainnet.base.org | \
  xargs -I {} cast code 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  --block {} --rpc-url https://mainnet.base.org

# Calculate transaction cost
TX_HASH="0x..."
GAS_USED=$(cast receipt $TX_HASH --json --rpc-url https://mainnet.base.org | jq -r .gasUsed)
GAS_PRICE=$(cast tx $TX_HASH --json --rpc-url https://mainnet.base.org | jq -r .gasPrice)
echo "scale=18; $GAS_USED * $GAS_PRICE / 1000000000000000000" | bc

# Watch pending transactions
cast rpc eth_subscribe newPendingTransactions --rpc-url wss://base-mainnet.g.alchemy.com/v2/$ALCHEMY_KEY

# Batch call multiple view functions
echo '["symbol","decimals","totalSupply"]' | jq -r '.[]' | \
  xargs -I {} cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "{}()" --rpc-url https://mainnet.base.org

# Find token holders from Transfer events
cast logs --from-block 0 --to-block latest \
  --address 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  "Transfer(address indexed, address indexed to, uint256)" \
  --rpc-url https://mainnet.base.org | \
  jq -r '.topics[2]' | sort -u

# Check if address is contract
[ $(cast code 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb --rpc-url https://mainnet.base.org | wc -c) -gt 3 ] && \
  echo "Contract" || echo "EOA"

# Get ERC20 token info as JSON
echo '{"symbol":"'$(cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 "symbol()" --rpc-url https://mainnet.base.org)'","decimals":'$(cast call 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 "decimals()" --rpc-url https://mainnet.base.org)'}' | jq

# Calculate time until block
CURRENT_BLOCK=$(cast block-number --rpc-url https://mainnet.base.org)
TARGET_BLOCK=10500000
BLOCKS_LEFT=$((TARGET_BLOCK - CURRENT_BLOCK))
SECONDS=$((BLOCKS_LEFT * 2))  # Base ~2s block time
echo "Approximately $((SECONDS / 3600)) hours until block $TARGET_BLOCK"
```

## Scripting with Cast

Use cast in shell scripts:

```bash
#!/bin/bash
# check-usdc-balance.sh

RPC="https://mainnet.base.org"
USDC="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
ADDRESS="$1"

if [ -z "$ADDRESS" ]; then
  echo "Usage: $0 <address>"
  exit 1
fi

# Get balance
BALANCE_RAW=$(cast call $USDC \
  "balanceOf(address)(uint256)" \
  $ADDRESS \
  --rpc-url $RPC)

# Convert to human-readable
BALANCE=$(cast --to-unit $BALANCE_RAW 6)

echo "USDC Balance for $ADDRESS: $BALANCE"

# Get USD value (assuming USDC = $1)
echo "Approximate USD value: \$$BALANCE"
```

Monitoring script:

```bash
#!/bin/bash
# monitor-transfers.sh

RPC="https://mainnet.base.org"
TOKEN="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
MIN_AMOUNT=1000000000  # 1000 USDC

LAST_BLOCK=$(cast block-number --rpc-url $RPC)

while true; do
  CURRENT_BLOCK=$(cast block-number --rpc-url $RPC)
  
  if [ $CURRENT_BLOCK -gt $LAST_BLOCK ]; then
    echo "Checking blocks $LAST_BLOCK to $CURRENT_BLOCK"
    
    cast logs \
      --from-block $LAST_BLOCK \
      --to-block $CURRENT_BLOCK \
      --address $TOKEN \
      "Transfer(address indexed, address indexed, uint256)" \
      --json \
      --rpc-url $RPC | \
      jq -r --arg min "$MIN_AMOUNT" \
        '.[] | select((.data | tonumber) > ($min | tonumber)) | 
        "Large transfer: " + .data + " from block " + .blockNumber'
    
    LAST_BLOCK=$CURRENT_BLOCK
  fi
  
  sleep 5
done
```

## Common Patterns

**Pattern 1: Batch Operations**

```bash
# Check balances for multiple addresses
for addr in 0x742d... 0x833d... 0x456e...; do
  balance=$(cast balance $addr --rpc-url https://mainnet.base.org)
  echo "$addr: $(cast --from-wei $balance) ETH"
done
```

**Pattern 2: Error Handling**

```bash
# Safely call with error handling
if result=$(cast call $TOKEN "balanceOf(address)(uint256)" $ADDR --rpc-url $RPC 2>&1); then
  echo "Balance: $result"
else
  echo "Error: $result"
  exit 1
fi
```

**Pattern 3: JSON Processing**

```bash
# Extract specific fields from transaction
cast tx $TX_HASH --json --rpc-url https://mainnet.base.org | \
  jq '{from: .from, to: .to, value: .value, gas: .gas}'
```

## Gotchas

**Gotcha 1: RPC Rate Limits**

Public RPCs have rate limits. Add delays:

```bash
for i in {1..100}; do
  cast call $TOKEN "balanceOf(address)" $ADDR --rpc-url $RPC
  sleep 0.1  # Avoid rate limiting
done
```

**Gotcha 2: Hex vs Decimal**

Cast often returns hex. Convert explicitly:

```bash
# Returns hex
cast call $TOKEN "decimals()(uint8)" --rpc-url $RPC
# Output: 0x06

# Convert to decimal
cast call $TOKEN "decimals()(uint8)" --rpc-url $RPC | cast --to-dec
# Output: 6
```

**Gotcha 3: String Encoding**

Strings require special handling:

```bash
# Get string return value
cast call $TOKEN "symbol()(string)" --rpc-url $RPC
# May return hex-encoded string

# Decode if needed
cast --to-ascii 0x555344430000000000000000000000000000000000000000000000000000000
```

**Gotcha 4: Transaction Confirmation**

Wait for confirmations on Base:

```bash
# Send and wait for 3 confirmations
TX=$(cast send $TOKEN "transfer(address,uint256)" $TO $AMOUNT \
  --private-key $PRIVATE_KEY \
  --rpc-url $RPC \
  --json | jq -r .transactionHash)

cast receipt $TX --confirmations 3 --rpc-url $RPC
```

**Gotcha 5: Block Time Variations**

Base block time averages ~2 seconds but varies:

```bash
# Check actual block time
BLOCK1=$(cast block 10000000 --json --rpc-url $RPC | jq -r .timestamp)
BLOCK2=$(cast block 10000001 --json --rpc-url $RPC | jq -r .timestamp)
echo "Block time: $((BLOCK2 - BLOCK1)) seconds"
```