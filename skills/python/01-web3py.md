# Web3.py for Base Development

## Overview

Web3.py is the standard Python library for interacting with Ethereum and EVM-compatible chains like Base. This lesson covers connecting to Base, reading contract state, sending transactions, filtering events, and working with ABIs.

## Installation and Setup

Install web3.py and dependencies:

```bash
pip install web3 python-dotenv
```

Basic connection to Base Mainnet:

```python
from web3 import Web3
import os
from dotenv import load_dotenv

load_dotenv()

# Connect to Base Mainnet
BASE_RPC_URL = "https://mainnet.base.org"
w3 = Web3(Web3.HTTPProvider(BASE_RPC_URL))

# Verify connection
if w3.is_connected():
    print(f"Connected to Base (Chain ID: {w3.eth.chain_id})")
    print(f"Latest block: {w3.eth.block_number}")
else:
    raise Exception("Failed to connect to Base")
```

## Reading Contract State

Read data from USDC contract on Base:

```python
from web3 import Web3
from eth_typing import Address

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

# USDC contract address on Base
USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"

# Minimal ERC-20 ABI
ERC20_ABI = [
    {
        "constant": True,
        "inputs": [],
        "name": "name",
        "outputs": [{"name": "", "type": "string"}],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "symbol",
        "outputs": [{"name": "", "type": "string"}],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "decimals",
        "outputs": [{"name": "", "type": "uint8"}],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [{"name": "account", "type": "address"}],
        "name": "balanceOf",
        "outputs": [{"name": "", "type": "uint256"}],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "totalSupply",
        "outputs": [{"name": "", "type": "uint256"}],
        "type": "function"
    }
]

# Create contract instance
usdc = w3.eth.contract(address=Web3.to_checksum_address(USDC_ADDRESS), abi=ERC20_ABI)

# Read contract state
name = usdc.functions.name().call()
symbol = usdc.functions.symbol().call()
decimals = usdc.functions.decimals().call()
total_supply = usdc.functions.totalSupply().call()

print(f"Token: {name} ({symbol})")
print(f"Decimals: {decimals}")
print(f"Total Supply: {total_supply / (10 ** decimals):,.2f} {symbol}")

# Check balance for specific address
address = "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432"  # Identity Registry
balance = usdc.functions.balanceOf(Web3.to_checksum_address(address)).call()
print(f"Balance of {address}: {balance / (10 ** decimals):,.2f} {symbol}")
```

## Sending Transactions

Send a transaction to transfer USDC:

```python
from web3 import Web3
from eth_account import Account
import os

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))
USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"

# Load private key from environment
private_key = os.getenv("PRIVATE_KEY")
account = Account.from_key(private_key)

# ERC-20 transfer ABI
TRANSFER_ABI = [{
    "constant": False,
    "inputs": [
        {"name": "to", "type": "address"},
        {"name": "value", "type": "uint256"}
    ],
    "name": "transfer",
    "outputs": [{"name": "", "type": "bool"}],
    "type": "function"
}]

usdc = w3.eth.contract(address=Web3.to_checksum_address(USDC_ADDRESS), abi=TRANSFER_ABI)

# Prepare transaction
recipient = "0x8004BAa17C55a88189AE136b182e5fdA19dE9b63"  # Reputation Registry
amount = Web3.to_wei(10, "mwei")  # 10 USDC (6 decimals)

transaction = usdc.functions.transfer(
    Web3.to_checksum_address(recipient),
    amount
).build_transaction({
    "from": account.address,
    "nonce": w3.eth.get_transaction_count(account.address),
    "gas": 100000,
    "maxFeePerGas": w3.eth.gas_price,
    "maxPriorityFeePerGas": w3.to_wei(1, "gwei"),
    "chainId": 8453
})

# Sign and send transaction
signed_txn = account.sign_transaction(transaction)
tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)

print(f"Transaction sent: {tx_hash.hex()}")

# Wait for confirmation
tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
print(f"Transaction confirmed in block {tx_receipt['blockNumber']}")
print(f"Gas used: {tx_receipt['gasUsed']}")
print(f"Status: {'Success' if tx_receipt['status'] == 1 else 'Failed'}")
```

## Event Filtering and Logs

Filter and process Transfer events from USDC:

```python
from web3 import Web3
from datetime import datetime, timedelta

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))
USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"

# Transfer event ABI
TRANSFER_EVENT_ABI = [{
    "anonymous": False,
    "inputs": [
        {"indexed": True, "name": "from", "type": "address"},
        {"indexed": True, "name": "to", "type": "address"},
        {"indexed": False, "name": "value", "type": "uint256"}
    ],
    "name": "Transfer",
    "type": "event"
}]

usdc = w3.eth.contract(address=Web3.to_checksum_address(USDC_ADDRESS), abi=TRANSFER_EVENT_ABI)

# Get events from last 1000 blocks
latest_block = w3.eth.block_number
from_block = latest_block - 1000

# Filter for all transfers
transfer_filter = usdc.events.Transfer.create_filter(
    fromBlock=from_block,
    toBlock="latest"
)

events = transfer_filter.get_all_entries()

print(f"Found {len(events)} Transfer events in blocks {from_block} to {latest_block}")

# Process events
total_volume = 0
for event in events[:10]:  # Show first 10
    from_addr = event["args"]["from"]
    to_addr = event["args"]["to"]
    value = event["args"]["value"]
    block_number = event["blockNumber"]
    tx_hash = event["transactionHash"].hex()
    
    total_volume += value
    
    print(f"Block {block_number}: {value / 1e6:.2f} USDC")
    print(f"  From: {from_addr}")
    print(f"  To: {to_addr}")
    print(f"  Tx: {tx_hash}")

print(f"\nTotal volume in sample: {total_volume / 1e6:,.2f} USDC")
```

## Filtering for Specific Addresses

Filter events for a specific address:

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))
USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"

# Aerodrome Router address
AERODROME_ROUTER = "0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43"

TRANSFER_EVENT_ABI = [{
    "anonymous": False,
    "inputs": [
        {"indexed": True, "name": "from", "type": "address"},
        {"indexed": True, "name": "to", "type": "address"},
        {"indexed": False, "name": "value", "type": "uint256"}
    ],
    "name": "Transfer",
    "type": "event"
}]

usdc = w3.eth.contract(address=Web3.to_checksum_address(USDC_ADDRESS), abi=TRANSFER_EVENT_ABI)

# Filter for transfers TO Aerodrome Router
transfer_filter = usdc.events.Transfer.create_filter(
    fromBlock=w3.eth.block_number - 500,
    toBlock="latest",
    argument_filters={"to": Web3.to_checksum_address(AERODROME_ROUTER)}
)

events = transfer_filter.get_all_entries()

print(f"USDC transfers TO Aerodrome Router: {len(events)}")

total_in = sum(e["args"]["value"] for e in events)
print(f"Total USDC sent to router: {total_in / 1e6:,.2f} USDC")

# Filter for transfers FROM Aerodrome Router
from_filter = usdc.events.Transfer.create_filter(
    fromBlock=w3.eth.block_number - 500,
    toBlock="latest",
    argument_filters={"from": Web3.to_checksum_address(AERODROME_ROUTER)}
)

events_from = from_filter.get_all_entries()
total_out = sum(e["args"]["value"] for e in events_from)

print(f"USDC transfers FROM Aerodrome Router: {len(events_from)}")
print(f"Total USDC sent from router: {total_out / 1e6:,.2f} USDC")
print(f"Net flow: {(total_in - total_out) / 1e6:,.2f} USDC")
```

## ABI Encoding and Decoding

Encode function calls and decode event data:

```python
from web3 import Web3
from eth_abi import encode, decode

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

# Encode a function call manually
function_signature = "transfer(address,uint256)"
function_selector = w3.keccak(text=function_signature)[:4]

recipient = "0x8004BAa17C55a88189AE136b182e5fdA19dE9b63"
amount = 1000000  # 1 USDC (6 decimals)

# Encode parameters
encoded_params = encode(
    ["address", "uint256"],
    [Web3.to_checksum_address(recipient), amount]
)

# Combine selector and params
encoded_call = function_selector + encoded_params
print(f"Encoded call data: 0x{encoded_call.hex()}")

# Decode event log data
event_signature = "Transfer(address,address,uint256)"
event_topic = w3.keccak(text=event_signature)
print(f"Transfer event topic: 0x{event_topic.hex()}")

# Simulate decoding event log
log_data = "0x00000000000000000000000000000000000000000000000000000000000f4240"  # 1000000
decoded_value = decode(["uint256"], bytes.fromhex(log_data[2:]))[0]
print(f"Decoded transfer amount: {decoded_value}")

# Using contract interface for encoding
TRANSFER_ABI = [{
    "constant": False,
    "inputs": [
        {"name": "to", "type": "address"},
        {"name": "value", "type": "uint256"}
    ],
    "name": "transfer",
    "outputs": [{"name": "", "type": "bool"}],
    "type": "function"
}]

contract = w3.eth.contract(abi=TRANSFER_ABI)
encoded = contract.encodeABI(
    fn_name="transfer",
    args=[Web3.to_checksum_address(recipient), amount]
)
print(f"Contract-encoded call: {encoded}")
```

## Common Patterns

**Batch Read Calls:**
Read multiple values efficiently:

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

addresses = [
    "0x8004A169FB4a3325136EB29fA0ceB6D2e539a432",
    "0x8004BAa17C55a88189AE136b182e5fdA19dE9b63",
    "0x0576a174D229E3cFA37253523E645A78A0C91B57"
]

USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
BALANCE_ABI = [{
    "constant": True,
    "inputs": [{"name": "account", "type": "address"}],
    "name": "balanceOf",
    "outputs": [{"name": "", "type": "uint256"}],
    "type": "function"
}]

usdc = w3.eth.contract(address=Web3.to_checksum_address(USDC_ADDRESS), abi=BALANCE_ABI)

balances = {}
for addr in addresses:
    balance = usdc.functions.balanceOf(Web3.to_checksum_address(addr)).call()
    balances[addr] = balance / 1e6
    print(f"{addr}: {balances[addr]:,.2f} USDC")
```

**Error Handling:**
Handle common Web3 errors:

```python
from web3 import Web3
from web3.exceptions import ContractLogicError, TransactionNotFound
import time

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))

def safe_get_transaction(tx_hash, max_retries=3):
    for attempt in range(max_retries):
        try:
            return w3.eth.get_transaction(tx_hash)
        except TransactionNotFound:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise

def safe_call_contract(contract, function_name, *args):
    try:
        result = getattr(contract.functions, function_name)(*args).call()
        return result
    except ContractLogicError as e:
        print(f"Contract reverted: {e}")
        return None
    except Exception as e:
        print(f"Error calling {function_name}: {e}")
        return None
```

## Gotchas

**Checksum Addresses:** Always use Web3.to_checksum_address() when passing addresses to contract functions. Lowercase addresses will cause errors.

**Gas Estimation:** Never rely solely on eth_estimateGas in production. Add a buffer (10-20%) to prevent out-of-gas errors.

**Event Filtering Limits:** RPC providers limit the block range for event queries. Base typically allows 2000-5000 blocks per query. Batch your requests for larger ranges.

**Nonce Management:** In high-throughput applications, track nonces locally to avoid conflicts. Don't rely on get_transaction_count() for every transaction.

**Block Reorganizations:** Recent blocks can be reorganized. Wait for 12+ confirmations for critical operations, especially on mainnet.

**Rate Limiting:** Public RPCs have rate limits. Implement exponential backoff and consider using paid RPC providers (Alchemy, Infura, QuickNode) for production.

**Decimal Precision:** ERC-20 tokens have different decimal places. USDC uses 6 decimals, not 18. Always check the decimals() function.

**Transaction Failures:** A transaction receipt with status=0 means the transaction reverted. Check the receipt status before assuming success.