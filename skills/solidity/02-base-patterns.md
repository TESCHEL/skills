# Base-Specific Patterns

## Overview

Base is an Optimistic Rollup (OP Stack) that provides full EVM compatibility with lower gas costs and faster block times (2 seconds). This lesson covers Base-specific patterns, optimizations, and system contracts that differ from Ethereum mainnet development.

## Base System Contracts

Base includes several system contracts at predefined addresses:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BaseSystemContracts {
    // L2 Cross Domain Messenger for L1<->L2 communication
    address constant L2_CROSS_DOMAIN_MESSENGER = 0x4200000000000000000000000000000000000007;
    
    // L2 Standard Bridge for token bridging
    address constant L2_STANDARD_BRIDGE = 0x4200000000000000000000000000000000000010;
    
    // L2 ERC721 Bridge
    address constant L2_ERC721_BRIDGE = 0x4200000000000000000000000000000000000014;
    
    // Optimism Mintable ERC20 Factory
    address constant OPTIMISM_MINTABLE_ERC20_FACTORY = 0x4200000000000000000000000000000000000012;
    
    // L1 Block Attributes (for L1 data)
    address constant L1_BLOCK_ATTRIBUTES = 0x4200000000000000000000000000000000000015;
    
    // Gas Price Oracle for L1 data fee calculation
    address constant GAS_PRICE_ORACLE = 0x420000000000000000000000000000000000000F;
    
    // Base Fee Vault
    address constant BASE_FEE_VAULT = 0x4200000000000000000000000000000000000019;
    
    // L1 Fee Vault
    address constant L1_FEE_VAULT = 0x420000000000000000000000000000000000001a;
}
```

## L1 to L2 Messaging

Cross-domain messaging enables communication between Ethereum L1 and Base L2:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ICrossDomainMessenger {
    function sendMessage(
        address _target,
        bytes calldata _message,
        uint32 _minGasLimit
    ) external payable;
    
    function xDomainMessageSender() external view returns (address);
}

contract L2MessageReceiver {
    address public constant CROSS_DOMAIN_MESSENGER = 0x4200000000000000000000000000000000000007;
    address public l1Counterpart;
    
    event MessageReceived(address indexed from, bytes data);
    
    error UnauthorizedCaller(address caller);
    error InvalidL1Sender(address sender);
    
    constructor(address _l1Counterpart) {
        l1Counterpart = _l1Counterpart;
    }
    
    modifier onlyFromL1Counterpart() {
        if (msg.sender != CROSS_DOMAIN_MESSENGER) {
            revert UnauthorizedCaller(msg.sender);
        }
        
        address l1Sender = ICrossDomainMessenger(CROSS_DOMAIN_MESSENGER).xDomainMessageSender();
        if (l1Sender != l1Counterpart) {
            revert InvalidL1Sender(l1Sender);
        }
        _;
    }
    
    function receiveMessage(bytes calldata data) external onlyFromL1Counterpart {
        emit MessageReceived(l1Counterpart, data);
        // Process message data
    }
}

contract L2MessageSender {
    address public constant CROSS_DOMAIN_MESSENGER = 0x4200000000000000000000000000000000000007;
    
    event MessageSent(address indexed to, bytes data, uint32 gasLimit);
    
    function sendToL1(address target, bytes calldata data, uint32 gasLimit) external {
        ICrossDomainMessenger(CROSS_DOMAIN_MESSENGER).sendMessage(
            target,
            data,
            gasLimit
        );
        
        emit MessageSent(target, data, gasLimit);
    }
}
```

## Gas Optimization for L2

Base transactions include both L2 execution gas and L1 data availability costs:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IGasPriceOracle {
    function getL1Fee(bytes memory _data) external view returns (uint256);
    function getL1GasUsed(bytes memory _data) external view returns (uint256);
}

contract GasOptimizedContract {
    address constant GAS_PRICE_ORACLE = 0x420000000000000000000000000000000000000F;
    
    // Pack storage to minimize slots
    struct User {
        uint96 balance;        // 96 bits
        uint96 lastUpdate;     // 96 bits
        uint64 rewardRate;     // 64 bits
        // Total: 256 bits = 1 slot
    }
    
    mapping(address => User) public users;
    
    // Use events for data instead of storage when possible
    event DataLogged(address indexed user, uint256 value, bytes32 dataHash);
    
    // Prefer calldata over memory for external functions
    function processData(uint[] calldata data) external {
        uint sum = 0;
        uint len = data.length;
        for (uint i = 0; i < len; i++) {
            sum += data[i];
        }
        emit DataLogged(msg.sender, sum, keccak256(abi.encode(data)));
    }
    
    // Batch operations to amortize fixed costs
    function batchUpdate(address[] calldata addresses, uint96[] calldata balances) external {
        require(addresses.length == balances.length, "Length mismatch");
        
        for (uint i = 0; i < addresses.length; i++) {
            users[addresses[i]].balance = balances[i];
            users[addresses[i]].lastUpdate = uint96(block.timestamp);
        }
    }
    
    // Estimate total gas cost including L1 data fee
    function estimateTotalCost(bytes calldata txData) external view returns (uint256) {
        uint256 l1Fee = IGasPriceOracle(GAS_PRICE_ORACLE).getL1Fee(txData);
        uint256 l2Gas = gasleft();
        uint256 l2Cost = l2Gas * tx.gasprice;
        return l1Fee + l2Cost;
    }
    
    // Use short strings and compress data
    function compressData(string calldata longString) external pure returns (bytes32) {
        return keccak256(bytes(longString));
    }
}
```

## Deploying on Base

Deterministic deployment using CREATE2 for predictable addresses:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract DeterministicDeployer {
    event Deployed(address indexed deployedAddress, bytes32 salt);
    
    error DeploymentFailed();
    
    function deploy(bytes memory bytecode, bytes32 salt) external returns (address) {
        address deployedAddress;
        
        assembly {
            deployedAddress := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        }
        
        if (deployedAddress == address(0)) {
            revert DeploymentFailed();
        }
        
        emit Deployed(deployedAddress, salt);
        return deployedAddress;
    }
    
    function computeAddress(bytes memory bytecode, bytes32 salt) external view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(bytecode)
            )
        );
        return address(uint160(uint256(hash)));
    }
}

contract SimpleContract {
    address public deployer;
    uint256 public value;
    
    constructor(uint256 _value) {
        deployer = msg.sender;
        value = _value;
    }
}
```

## Base Block Properties

Base has faster block times and different block properties:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BaseBlockProperties {
    // Base produces blocks every 2 seconds
    uint256 public constant BASE_BLOCK_TIME = 2 seconds;
    
    // Use block.timestamp instead of block.number for time logic
    function isExpired(uint256 expiryTimestamp) public view returns (bool) {
        return block.timestamp >= expiryTimestamp;
    }
    
    // Calculate time-based delays
    function getExpiryTimestamp(uint256 delayInSeconds) public view returns (uint256) {
        return block.timestamp + delayInSeconds;
    }
    
    // Example: Time-locked function
    mapping(bytes32 => uint256) public timelocks;
    
    function schedule(bytes32 actionId, uint256 delayInSeconds) external {
        timelocks[actionId] = block.timestamp + delayInSeconds;
    }
    
    function execute(bytes32 actionId) external {
        require(block.timestamp >= timelocks[actionId], "Too early");
        require(timelocks[actionId] != 0, "Not scheduled");
        
        delete timelocks[actionId];
        // Execute action
    }
}
```

## Bridged Token Pattern

Pattern for tokens bridged from L1 to Base:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

interface IL2StandardBridge {
    function withdraw(
        address _l2Token,
        uint256 _amount,
        uint32 _minGasLimit,
        bytes calldata _extraData
    ) external payable;
}

contract L2BridgedToken is ERC20, Ownable {
    address public constant L2_BRIDGE = 0x4200000000000000000000000000000000000010;
    address public immutable l1Token;
    
    event BridgeMint(address indexed account, uint256 amount);
    event BridgeBurn(address indexed account, uint256 amount);
    
    error OnlyBridge(address caller);
    
    constructor(
        string memory name,
        string memory symbol,
        address _l1Token
    ) ERC20(name, symbol) Ownable(msg.sender) {
        l1Token = _l1Token;
    }
    
    modifier onlyBridge() {
        if (msg.sender != L2_BRIDGE) {
            revert OnlyBridge(msg.sender);
        }
        _;
    }
    
    function mint(address account, uint256 amount) external onlyBridge {
        _mint(account, amount);
        emit BridgeMint(account, amount);
    }
    
    function burn(address account, uint256 amount) external onlyBridge {
        _burn(account, amount);
        emit BridgeBurn(account, amount);
    }
    
    function withdrawToL1(uint256 amount, uint32 minGasLimit) external {
        _burn(msg.sender, amount);
        
        IL2StandardBridge(L2_BRIDGE).withdraw(
            address(this),
            amount,
            minGasLimit,
            ""
        );
    }
}
```

## Common Patterns

**Efficient Event Logging:**

```solidity
contract EfficientEvents {
    // Use indexed parameters for filtering (max 3)
    event Transfer(address indexed from, address indexed to, uint256 amount);
    
    // Store large data off-chain, emit hash
    event DataStored(address indexed user, bytes32 indexed dataHash, uint256 timestamp);
    
    function storeData(bytes calldata data) external {
        bytes32 dataHash = keccak256(data);
        emit DataStored(msg.sender, dataHash, block.timestamp);
    }
}
```

**Multicall Pattern:**

```solidity
contract Multicall {
    function aggregate(bytes[] calldata calls) external returns (bytes[] memory results) {
        results = new bytes[](calls.length);
        for (uint i = 0; i < calls.length; i++) {
            (bool success, bytes memory result) = address(this).delegatecall(calls[i]);
            require(success, "Call failed");
            results[i] = result;
        }
    }
}
```

## Gotchas

- **L1 Data Costs**: Transaction data is posted to L1, so minimize calldata size.
- **Block Time Assumptions**: Base blocks are 2 seconds, not 12-15 like Ethereum.
- **Cross-Domain Messages**: L1->L2 messages are near-instant, but L2->L1 requires a 7-day challenge period.
- **Gas Price Oracle**: L1 data fees vary with Ethereum L1 gas prices.
- **Reorg Resistance**: Base has very low reorg risk due to fast finality.
- **System Contract Addresses**: Always use the correct predeploy addresses (0x4200...).