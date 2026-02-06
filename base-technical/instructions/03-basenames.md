# Basenames: Programmatic Registration on Base

## Overview
Basenames are Base's ENS subnames for the `base.eth` domain. Human-readable addresses (e.g., `alice.base.eth`) that map to Ethereum addresses and onchain resources.

## Contract Addresses (Base Mainnet)
- **Registry:** `0x627...dadd`
- **BaseRegistrar:** `0xd1a...5d67`
- **RegistrarController:** `0x1f7...d3fc`
- **Price Oracle:** `0x3f9...c4bc`
- **L2Resolver:** deployed (handles text records on Base)
- **ReverseRegistrar:** deployed (address â name mapping)

## Registration Function
```solidity
register(bytes32 label, address owner, uint duration, bytes[] calldata data)
```
Called on RegistrarController. Must include ETH payment matching Price Oracle output.

**Duration:** 1 year = 31,556,952 seconds

## Price Structure (Annual)
| Name Length | Annual Cost |
|-------------|------------|
| 3 characters | 0.1 ETH |
| 4 characters | 0.01 ETH |
| 5-9 characters | 0.001 ETH |
| 10+ characters | 0.0001 ETH |

**"baised" = 6 characters = 0.001 ETH/year**

## Free Registration Eligibility
- Coinbase Verified users
- Base Mainnet Builder NFT holders
- Summer Pass Level 3 NFT owners
- Base Builders program participants
- base.eth NFT owners

## Setting Text Records
Via L2Resolver contract:
```solidity
setText(namehash, key, value)
```
Available keys: `avatar`, `description`, `url`, `x402`, `twitter`, `com.twitter`, `email`, etc.

## Reverse Resolution
Via ReverseRegistrar:
```solidity
setName(address, name)
```
Maps address back to Basename. Essential for other contracts to identify the agent.

## JavaScript SDK Example
```javascript
import { NameKit, networks } from '@ensdomains/namekit';
import { ethers } from 'ethers';

const provider = new ethers.providers.JsonRpcProvider('https://mainnet.base.org');
const wallet = new ethers.Wallet('0xPRIVATE_KEY', provider);
const namekit = new NameKit({ chainId: networks.base.id, provider });

const price = await namekit.getRentPrice('baised');

const tx = await namekit.register({
  name: 'baised',
  owner: wallet.address,
  duration: 31556952,
  resolver: namekit.resolverAddress,
  records: {
    text: {
      avatar: 'ipfs://QmAvatarHash',
      description: 'Ecosystem intelligence agent',
      url: 'https://baised.ai'
    },
    address: { 60: wallet.address }
  }
});
await tx.wait();
```

## BAiSED Registration Plan
1. Register `baised.base.eth` (0.001 ETH for 1 year)
2. Set text records: avatar, description, url, x402 endpoint, twitter handle
3. Set reverse resolution for smart wallet address
4. Include Basename in ERC-8004 registration file (`nameService` field)
5. Set calendar reminder for annual renewal