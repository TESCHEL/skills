---
name: foundry
description: Foundry toolkit for Solidity smart contract development, testing, and deployment on Base
trigger_phrases:
  - "use foundry to test"
  - "deploy with forge"
  - "set up foundry project"
  - "foundry testing framework"
  - "forge script deployment"
  - "cast command for"
version: 1.0.0
---

# Foundry Toolkit for Base Development

## Guidelines

Foundry is a blazing fast, portable, and modular toolkit for Ethereum application development written in Rust. Use this skill when you need to:

- Initialize and configure Solidity projects with modern tooling
- Write comprehensive test suites with fuzzing and invariant testing
- Deploy smart contracts to Base mainnet or testnet with scripts
- Interact with deployed contracts using command-line tools
- Perform advanced testing scenarios including fork testing against live Base state
- Debug contract behavior with detailed trace outputs
- Verify contracts on Basescan automatically after deployment

Foundry consists of four main tools:

1. **Forge**: Ethereum testing framework (like Hardhat, Truffle, DappTools)
2. **Cast**: Swiss army knife for interacting with EVM smart contracts, sending transactions, and getting chain data
3. **Anvil**: Local Ethereum node, similar to Ganache or Hardhat Network
4. **Chisel**: Fast, interactive Solidity REPL

Choose Foundry over Hardhat when:
- You need extremely fast test execution (10-100x faster)
- You prefer writing tests in Solidity rather than JavaScript
- You want built-in fuzzing and invariant testing capabilities
- You need powerful scripting for complex deployment workflows
- You value reproducible builds and deterministic deployments

Foundry integrates seamlessly with Base blockchain:
- Native support for Base RPC endpoints
- Automatic Basescan verification
- Fork testing against Base mainnet state
- Gas optimization for L2 deployment costs

## Examples

**Example 1: Initialize and Test a Project**

Trigger: "Set up a foundry project and write tests for an ERC20 token"

Action:
```bash
forge init my-token
cd my-token

# Install OpenZeppelin contracts
forge install OpenZeppelin/openzeppelin-contracts

# Create token contract
cat > src/MyToken.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor() ERC20("MyToken", "MTK") {
        _mint(msg.sender, 1000000 * 10 ** decimals());
    }
}
EOF

# Create test file
cat > test/MyToken.t.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken token;
    address alice = address(0x1);
    address bob = address(0x2);

    function setUp() public {
        token = new MyToken();
    }

    function testInitialSupply() public {
        assertEq(token.totalSupply(), 1000000 * 10 ** 18);
    }

    function testTransfer() public {
        uint256 amount = 1000 * 10 ** 18;
        token.transfer(alice, amount);
        assertEq(token.balanceOf(alice), amount);
    }

    function testFuzzTransfer(uint256 amount) public {
        vm.assume(amount <= token.totalSupply());
        token.transfer(alice, amount);
        assertEq(token.balanceOf(alice), amount);
    }
}
EOF

# Run tests
forge test -vv
```

**Example 2: Deploy to Base with Verification**

Trigger: "Deploy my contract to Base mainnet and verify it"

Action:
```bash
# Create deployment script
cat > script/Deploy.s.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract DeployScript is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);

        MyToken token = new MyToken();
        console.log("Token deployed to:", address(token));

        vm.stopBroadcast();
    }
}
EOF

# Deploy and verify
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url https://mainnet.base.org \
  --broadcast \
  --verify \
  --etherscan-api-key $BASESCAN_API_KEY
```

**Example 3: Fork Test Against Base Mainnet**

Trigger: "Test my contract interactions with USDC on Base"

Action:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract USDCForkTest is Test {
    IERC20 constant USDC = IERC20(0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913);
    address constant USDC_WHALE = 0xcdac0d6c6c59727a65f871236188350531885c43;
    address alice = address(0x1);

    function setUp() public {
        // Fork Base mainnet
        vm.createSelectFork("https://mainnet.base.org");
    }

    function testTransferUSDC() public {
        // Impersonate whale account
        vm.startPrank(USDC_WHALE);
        
        uint256 amount = 1000 * 10 ** 6; // 1000 USDC
        USDC.transfer(alice, amount);
        
        vm.stopPrank();
        
        assertEq(USDC.balanceOf(alice), amount);
    }
}
```

## Resources

- Foundry Book: https://book.getfoundry.sh/
- Foundry GitHub: https://github.com/foundry-rs/foundry
- Base RPC: https://mainnet.base.org
- Base Testnet (Sepolia): https://sepolia.base.org
- Basescan: https://basescan.org
- Basescan API: https://docs.basescan.org/
- Forge Standard Library: https://github.com/foundry-rs/forge-std
- Solidity by Example (Foundry): https://solidity-by-example.org/foundry/

## Cross-References

- **solidity**: Core language for writing contracts tested and deployed with Foundry
- **hardhat**: Alternative development framework -- compare trade-offs
- **git**: Version control for Foundry projects; forge init creates git repos
- **basescan-verification**: Automated verification via forge verify-contract
- **testing**: General testing principles applied in Foundry test suites
- **deployment**: Deployment strategies implemented with forge script
- **viem**: TypeScript client for interacting with contracts deployed via Foundry