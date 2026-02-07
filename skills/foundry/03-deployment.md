# Foundry Deployment and Verification

## Overview

This lesson covers deploying smart contracts to Base using forge create and forge script, verifying contracts on Basescan, implementing deployment scripts with proper access control, and using CREATE2 for deterministic deployments.

## Simple Deployment with forge create

Deploy contracts directly from the command line:

```bash
# Basic deployment to Base
forge create src/MyToken.sol:MyToken \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY

# Deploy with constructor arguments
forge create src/MyToken.sol:MyToken \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY \
  --constructor-args "MyToken" "MTK" 1000000

# Deploy with verification
forge create src/MyToken.sol:MyToken \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY \
  --verify \
  --etherscan-api-key $BASESCAN_API_KEY

# Deploy to Base Sepolia testnet
forge create src/MyToken.sol:MyToken \
  --rpc-url https://sepolia.base.org \
  --private-key $PRIVATE_KEY \
  --constructor-args "TestToken" "TEST" 1000000 \
  --verify \
  --etherscan-api-key $BASESCAN_API_KEY

# Deploy with specific gas price (important for Base)
forge create src/MyToken.sol:MyToken \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY \
  --gas-price 1000000000  # 1 gwei

# Deploy with gas limit
forge create src/MyToken.sol:MyToken \
  --rpc-url https://mainnet.base.org \
  --private-key $PRIVATE_KEY \
  --gas-limit 3000000
```

Using named RPC endpoints from foundry.toml:

```bash
# Reference configured RPC
forge create src/MyToken.sol:MyToken \
  --rpc-url base \
  --private-key $PRIVATE_KEY
```

## Deployment Scripts

Forge scripts provide reproducible, version-controlled deployments:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract DeployToken is Script {
    function run() external {
        // Load private key from environment
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        // Start recording transactions
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy contract
        MyToken token = new MyToken("MyToken", "MTK", 1000000e18);
        
        console.log("Token deployed to:", address(token));
        console.log("Deployer:", vm.addr(deployerPrivateKey));
        
        // Stop recording
        vm.stopBroadcast();
    }
}
```

Execute deployment script:

```bash
# Dry run (simulation)
forge script script/DeployToken.s.sol:DeployToken \
  --rpc-url https://mainnet.base.org

# Execute deployment
forge script script/DeployToken.s.sol:DeployToken \
  --rpc-url https://mainnet.base.org \
  --broadcast \
  --verify \
  --etherscan-api-key $BASESCAN_API_KEY

# Deploy with specific wallet
forge script script/DeployToken.s.sol:DeployToken \
  --rpc-url https://mainnet.base.org \
  --broadcast \
  --private-key $PRIVATE_KEY

# Deploy to testnet
forge script script/DeployToken.s.sol:DeployToken \
  --rpc-url https://sepolia.base.org \
  --broadcast
```

Deployment artifacts are saved to:
```
broadcast/
  DeployToken.s.sol/
    8453/                    # Base chain ID
      run-latest.json        # Latest deployment
      run-1234567890.json    # Timestamped deployments
```

## Multi-Contract Deployment

Deploy multiple contracts with dependencies:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/Token.sol";
import "../src/Vault.sol";
import "../src/Governor.sol";

contract DeployProtocol is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy token
        Token token = new Token("Protocol Token", "PTK");
        console.log("Token deployed:", address(token));
        
        // Deploy vault with token address
        Vault vault = new Vault(address(token));
        console.log("Vault deployed:", address(vault));
        
        // Deploy governor
        Governor governor = new Governor(
            address(token),
            address(vault),
            deployer  // Initial owner
        );
        console.log("Governor deployed:", address(governor));
        
        // Transfer vault ownership to governor
        vault.transferOwnership(address(governor));
        console.log("Vault ownership transferred to Governor");
        
        // Initialize with parameters
        governor.initialize({
            votingDelay: 1 days,
            votingPeriod: 7 days,
            proposalThreshold: 100000e18
        });
        console.log("Governor initialized");
        
        vm.stopBroadcast();
        
        // Log summary
        console.log("\n=== Deployment Summary ===");
        console.log("Deployer:", deployer);
        console.log("Token:", address(token));
        console.log("Vault:", address(vault));
        console.log("Governor:", address(governor));
        console.log("Chain ID:", block.chainid);
    }
}
```

## Configuration-Based Deployment

Use environment-specific configurations:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyProtocol.sol";

contract DeployConfigurable is Script {
    struct Config {
        address usdc;
        address weth;
        address router;
        uint256 fee;
    }
    
    function getConfig(uint256 chainId) internal pure returns (Config memory) {
        if (chainId == 8453) {
            // Base mainnet
            return Config({
                usdc: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913,
                weth: 0x4200000000000000000000000000000000000006,
                router: 0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43,
                fee: 30  // 0.3%
            });
        } else if (chainId == 84532) {
            // Base Sepolia
            return Config({
                usdc: 0x036CbD53842c5426634e7929541eC2318f3dCF7e,
                weth: 0x4200000000000000000000000000000000000006,
                router: 0x1689E7B1F10000AE47eBfE339a4f69dECd19F602,
                fee: 30
            });
        } else {
            revert("Unsupported chain");
        }
    }
    
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        Config memory config = getConfig(block.chainid);
        
        vm.startBroadcast(deployerPrivateKey);
        
        MyProtocol protocol = new MyProtocol(
            config.usdc,
            config.weth,
            config.router,
            config.fee
        );
        
        console.log("Protocol deployed to:", address(protocol));
        console.log("Chain ID:", block.chainid);
        
        vm.stopBroadcast();
    }
}
```

## Verification on Basescan

Verify contracts after deployment:

```bash
# Verify during deployment
forge script script/Deploy.s.sol:DeployToken \
  --rpc-url https://mainnet.base.org \
  --broadcast \
  --verify \
  --etherscan-api-key $BASESCAN_API_KEY

# Verify existing contract
forge verify-contract \
  0x1234567890123456789012345678901234567890 \
  src/MyToken.sol:MyToken \
  --chain-id 8453 \
  --etherscan-api-key $BASESCAN_API_KEY

# Verify with constructor arguments
forge verify-contract \
  0x1234567890123456789012345678901234567890 \
  src/MyToken.sol:MyToken \
  --chain-id 8453 \
  --etherscan-api-key $BASESCAN_API_KEY \
  --constructor-args $(cast abi-encode "constructor(string,string,uint256)" "MyToken" "MTK" 1000000)

# Verify with compiler version
forge verify-contract \
  0x1234567890123456789012345678901234567890 \
  src/MyToken.sol:MyToken \
  --chain-id 8453 \
  --etherscan-api-key $BASESCAN_API_KEY \
  --compiler-version v0.8.20+commit.a1b79de6

# Verify via Sourcify (alternative)
forge verify-contract \
  0x1234567890123456789012345678901234567890 \
  src/MyToken.sol:MyToken \
  --chain-id 8453 \
  --verifier sourcify
```

Configure Basescan in foundry.toml:

```toml
[etherscan]
base = { 
  key = "${BASESCAN_API_KEY}",
  url = "https://api.basescan.org/api"
}
base_sepolia = { 
  key = "${BASESCAN_API_KEY}",
  url = "https://api-sepolia.basescan.org/api" 
}
```

## CREATE2 Deterministic Deployment

Deploy contracts to deterministic addresses:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract DeployDeterministic is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        // Salt for CREATE2
        bytes32 salt = bytes32(uint256(0x1234));
        
        // Compute address before deployment
        bytes memory bytecode = abi.encodePacked(
            type(MyToken).creationCode,
            abi.encode("MyToken", "MTK", 1000000e18)
        );
        
        address predictedAddress = vm.computeCreate2Address(
            salt,
            keccak256(bytecode),
            deployer
        );
        
        console.log("Predicted address:", predictedAddress);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy with CREATE2
        MyToken token = new MyToken{salt: salt}("MyToken", "MTK", 1000000e18);
        
        vm.stopBroadcast();
        
        require(address(token) == predictedAddress, "Address mismatch");
        console.log("Token deployed to:", address(token));
    }
}
```

Using a CREATE2 factory for cross-chain deterministic deployment:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";

interface ICREATE2Factory {
    function deploy(bytes32 salt, bytes memory bytecode) external returns (address);
    function computeAddress(bytes32 salt, bytes32 bytecodeHash) external view returns (address);
}

contract DeployCrossChain is Script {
    // Canonical CREATE2 factory (same address on all chains)
    ICREATE2Factory constant FACTORY = ICREATE2Factory(0x4e59b44847b379578588920cA78FbF26c0B4956C);
    
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        bytes32 salt = keccak256("MyToken.v1");
        bytes memory bytecode = abi.encodePacked(
            type(MyToken).creationCode,
            abi.encode("MyToken", "MTK", 1000000e18)
        );
        
        address predictedAddress = FACTORY.computeAddress(
            salt,
            keccak256(bytecode)
        );
        
        console.log("Will deploy to:", predictedAddress);
        console.log("Same address on all chains!");
        
        vm.startBroadcast(deployerPrivateKey);
        
        address deployed = FACTORY.deploy(salt, bytecode);
        require(deployed == predictedAddress, "Address mismatch");
        
        vm.stopBroadcast();
        
        console.log("Deployed to:", deployed);
    }
}
```

## Upgradeability

Deploy upgradeable contracts using proxies:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import "../src/MyTokenUpgradeable.sol";

contract DeployUpgradeable is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy implementation
        MyTokenUpgradeable implementation = new MyTokenUpgradeable();
        console.log("Implementation:", address(implementation));
        
        // Encode initializer call
        bytes memory initData = abi.encodeWithSelector(
            MyTokenUpgradeable.initialize.selector,
            "MyToken",
            "MTK",
            deployer
        );
        
        // Deploy proxy
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(implementation),
            initData
        );
        console.log("Proxy:", address(proxy));
        
        vm.stopBroadcast();
        
        // Interact through proxy
        MyTokenUpgradeable token = MyTokenUpgradeable(address(proxy));
        console.log("Token name:", token.name());
    }
}
```

## Post-Deployment Actions

Perform actions after deployment:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyProtocol.sol";

contract DeployAndConfigure is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy
        MyProtocol protocol = new MyProtocol();
        console.log("Protocol deployed:", address(protocol));
        
        // Configure
        protocol.setParameter("feeRate", 30);
        protocol.setParameter("maxSlippage", 100);
        
        // Grant roles
        protocol.grantRole(protocol.OPERATOR_ROLE(), 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb);
        
        // Whitelist addresses
        address[] memory whitelist = new address[](3);
        whitelist[0] = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913;  // USDC
        whitelist[1] = 0x4200000000000000000000000000000000000006;  // WETH
        whitelist[2] = 0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43;  // Aerodrome Router
        
        for (uint i = 0; i < whitelist.length; i++) {
            protocol.addToWhitelist(whitelist[i]);
        }
        
        // Transfer ownership
        address multisig = vm.envAddress("MULTISIG_ADDRESS");
        protocol.transferOwnership(multisig);
        console.log("Ownership transferred to:", multisig);
        
        vm.stopBroadcast();
    }
}
```

## Multi-Chain Deployment

Deploy to multiple chains in one script:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract DeployMultiChain is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        // Deploy to Base
        vm.createSelectFork("https://mainnet.base.org");
        vm.startBroadcast(deployerPrivateKey);
        MyToken baseToken = new MyToken("MyToken Base", "MTK", 1000000e18);
        vm.stopBroadcast();
        console.log("Base deployment:", address(baseToken));
        
        // Deploy to Base Sepolia
        vm.createSelectFork("https://sepolia.base.org");
        vm.startBroadcast(deployerPrivateKey);
        MyToken sepoliaToken = new MyToken("MyToken Sepolia", "MTK", 1000000e18);
        vm.stopBroadcast();
        console.log("Sepolia deployment:", address(sepoliaToken));
    }
}
```

Execute multi-chain deployment:

```bash
# Deploy to Base
forge script script/DeployMultiChain.s.sol:DeployMultiChain \
  --rpc-url https://mainnet.base.org \
  --broadcast

# Deploy to Base Sepolia
forge script script/DeployMultiChain.s.sol:DeployMultiChain \
  --rpc-url https://sepolia.base.org \
  --broadcast
```

## Common Patterns

**Pattern 1: Deployment Registry**

Track deployments across chains:

```solidity
contract DeploymentRegistry is Script {
    struct Deployment {
        address token;
        address vault;
        address governor;
        uint256 timestamp;
    }
    
    mapping(uint256 => Deployment) public deployments;
    
    function saveDeployment(Deployment memory deployment) internal {
        deployments[block.chainid] = deployment;
        
        string memory json = string.concat(
            '{"chainId":', vm.toString(block.chainid),
            ',"token":"', vm.toString(deployment.token),
            '","vault":"', vm.toString(deployment.vault),
            '","governor":"', vm.toString(deployment.governor),
            '","timestamp":', vm.toString(deployment.timestamp),
            '}'
        );
        
        string memory file = string.concat(
            "deployments/",
            vm.toString(block.chainid),
            ".json"
        );
        
        vm.writeFile(file, json);
    }
}
```

**Pattern 2: Deployment with Timelock**

```solidity
function run() external {
    vm.startBroadcast(deployerPrivateKey);
    
    // Deploy with immediate ownership transfer
    MyToken token = new MyToken();
    
    // Transfer to timelock for governance
    token.transferOwnership(TIMELOCK_ADDRESS);
    
    vm.stopBroadcast();
}
```

## Gotchas

**Gotcha 1: Nonce Management**

Ensure correct nonce when deploying multiple contracts:

```solidity
// Deploy order matters for address prediction
Token token = new Token();      // Nonce 0
Vault vault = new Vault();      // Nonce 1
Governor gov = new Governor();  // Nonce 2
```

**Gotcha 2: Gas Price on Base**

Base has different gas dynamics than Ethereum:

```bash
# Check current gas price
cast gas-price --rpc-url https://mainnet.base.org

# Deploy with appropriate gas price
forge script script/Deploy.s.sol \
  --rpc-url https://mainnet.base.org \
  --broadcast \
  --legacy  # Use legacy transactions if needed
```

**Gotcha 3: Constructor Arguments Encoding**

Correctly encode complex constructor arguments:

```bash
# For arrays and structs
cast abi-encode "constructor(address[],uint256[])" "[0x...,0x...]" "[100,200]"
```

**Gotcha 4: Private Key Security**

Never commit private keys:

```bash
# Use hardware wallet or keystore
forge script script/Deploy.s.sol \
  --rpc-url base \
  --ledger \
  --sender 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb
```

**Gotcha 5: Verification Delay**

Wait for block confirmation before verification:

```bash
# Deploy
forge script script/Deploy.s.sol --rpc-url base --broadcast

# Wait ~30 seconds for block propagation
sleep 30

# Then verify
forge verify-contract <address> <contract> --chain-id 8453
```