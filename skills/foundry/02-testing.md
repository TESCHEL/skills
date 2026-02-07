# Foundry Testing Framework

## Overview

Foundry provides a powerful testing framework written in Solidity. This lesson covers writing tests, using assertions, fuzzing, invariant testing, fork testing against Base mainnet, and leveraging cheatcodes for advanced test scenarios.

## Test File Structure

Foundry tests are Solidity contracts that inherit from `forge-std/Test.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;
    address alice = address(0x1);
    address bob = address(0x2);

    // Runs before each test
    function setUp() public {
        counter = new Counter();
        counter.setNumber(0);
    }

    // Test functions must start with "test"
    function testIncrement() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    // Test expected failures with "testFail"
    function testFailUnauthorized() public {
        vm.prank(alice);
        // This should revert
        counter.setNumber(100);
    }

    // Test with expectRevert
    function testRevertUnauthorized() public {
        vm.expectRevert("Unauthorized");
        vm.prank(alice);
        counter.setNumber(100);
    }
}
```

Test naming conventions:
- `test<FunctionName>`: Basic test
- `testFail<FunctionName>`: Expects test to revert
- `testFuzz_<FunctionName>`: Fuzz test
- `invariant_<Description>`: Invariant test

## Running Tests

Execute tests with forge test:

```bash
# Run all tests
forge test

# Run with verbosity (show logs)
forge test -vv

# Run with max verbosity (show traces)
forge test -vvvv

# Run specific test file
forge test --match-path test/Counter.t.sol

# Run specific test function
forge test --match-test testIncrement

# Run tests matching pattern
forge test --match-contract CounterTest

# Show gas report
forge test --gas-report

# Run with coverage
forge coverage
```

Verbosity levels:
- `-v`: Show test results
- `-vv`: Show logs (console.log)
- `-vvv`: Show traces for failing tests
- `-vvvv`: Show traces for all tests
- `-vvvvv`: Show detailed traces

## Assertions

Forge provides comprehensive assertion functions:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract AssertionTest is Test {
    function testAssertions() public {
        // Equality
        assertEq(1, 1);
        assertEq("hello", "hello");
        assertEq(address(0x1), address(0x1));
        
        // Inequality
        assertTrue(1 != 2);
        assertFalse(1 == 2);
        
        // Greater/Less than
        assertGt(2, 1);  // greater than
        assertGe(2, 2);  // greater than or equal
        assertLt(1, 2);  // less than
        assertLe(2, 2);  // less than or equal
        
        // Approximate equality for decimals
        assertApproxEqAbs(100, 101, 1);  // Within 1
        assertApproxEqRel(100, 105, 0.05e18);  // Within 5%
        
        // Memory equality
        bytes memory a = "test";
        bytes memory b = "test";
        assertEq(a, b);
        
        // Array equality
        uint256[] memory arr1 = new uint256[](2);
        arr1[0] = 1;
        arr1[1] = 2;
        uint256[] memory arr2 = new uint256[](2);
        arr2[0] = 1;
        arr2[1] = 2;
        assertEq(arr1, arr2);
    }
}
```

## Cheatcodes

Foundry cheatcodes manipulate blockchain state for testing:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract CheatcodeTest is Test {
    ERC20 token;
    address alice = address(0x1);
    address bob = address(0x2);

    function setUp() public {
        token = new ERC20("Test", "TST");
    }

    function testPrank() public {
        // Next call from alice
        vm.prank(alice);
        token.approve(bob, 100);
        
        // Multiple calls from alice
        vm.startPrank(alice);
        token.approve(bob, 100);
        token.transfer(bob, 50);
        vm.stopPrank();
    }

    function testDeal() public {
        // Give alice 100 ETH
        vm.deal(alice, 100 ether);
        assertEq(alice.balance, 100 ether);
        
        // Set token balance (requires special handling)
        deal(address(token), alice, 1000e18);
        assertEq(token.balanceOf(alice), 1000e18);
    }

    function testWarp() public {
        // Set block.timestamp
        vm.warp(1704067200);  // Jan 1, 2024
        assertEq(block.timestamp, 1704067200);
        
        // Skip forward 1 day
        skip(1 days);
        assertEq(block.timestamp, 1704067200 + 1 days);
        
        // Rewind 1 hour
        rewind(1 hours);
    }

    function testRoll() public {
        // Set block.number
        vm.roll(1000000);
        assertEq(block.number, 1000000);
    }

    function testFee() public {
        // Set block.basefee (important for Base)
        vm.fee(100 gwei);
        assertEq(block.basefee, 100 gwei);
    }

    function testExpectRevert() public {
        // Expect next call to revert
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 1000);
        
        // Expect revert with custom error
        vm.expectRevert(abi.encodeWithSignature("InsufficientBalance(uint256,uint256)", 0, 1000));
        token.transfer(bob, 1000);
    }

    function testExpectEmit() public {
        // Expect Transfer event
        vm.expectEmit(true, true, false, true);
        emit IERC20.Transfer(address(this), alice, 100);
        token.transfer(alice, 100);
    }

    function testMockCall() public {
        // Mock token.balanceOf(alice) to return 9999
        vm.mockCall(
            address(token),
            abi.encodeWithSelector(token.balanceOf.selector, alice),
            abi.encode(9999)
        );
        
        assertEq(token.balanceOf(alice), 9999);
        
        // Clear mock
        vm.clearMockedCalls();
    }

    function testSnapshot() public {
        // Take snapshot
        uint256 snapshot = vm.snapshot();
        
        token.transfer(alice, 100);
        assertEq(token.balanceOf(alice), 100);
        
        // Revert to snapshot
        vm.revertTo(snapshot);
        assertEq(token.balanceOf(alice), 0);
    }
}
```

Common cheatcodes:
- `vm.prank(address)`: Next call from address
- `vm.startPrank(address)`: All calls from address until stopPrank
- `vm.deal(address, uint)`: Set ETH balance
- `vm.warp(uint)`: Set block.timestamp
- `vm.roll(uint)`: Set block.number
- `vm.fee(uint)`: Set block.basefee
- `vm.expectRevert()`: Expect next call to revert
- `vm.expectEmit()`: Expect event emission
- `vm.mockCall()`: Mock contract call
- `vm.etch(address, bytes)`: Set contract code

## Fuzzing

Foundry automatically fuzzes test inputs:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract FuzzTest is Test {
    // Foundry runs this with random uint256 values
    function testFuzz_Add(uint256 a, uint256 b) public {
        vm.assume(a < type(uint256).max - b);  // Prevent overflow
        unchecked {
            assertEq(a + b, b + a);  // Commutative
            assertGe(a + b, a);      // Monotonic
        }
    }

    // Fuzz with multiple parameters
    function testFuzz_Transfer(address from, address to, uint256 amount) public {
        // Filter inputs
        vm.assume(from != address(0));
        vm.assume(to != address(0));
        vm.assume(from != to);
        vm.assume(amount > 0 && amount < 1000000e18);
        
        // Test logic
        vm.prank(from);
        // ... transfer implementation
    }

    // Bound inputs to specific range
    function testFuzz_BoundedTransfer(uint256 amount) public {
        // Bound amount between 1 and 1000
        amount = bound(amount, 1, 1000);
        assertGe(amount, 1);
        assertLe(amount, 1000);
    }

    // Configure fuzz runs in foundry.toml:
    // [profile.default]
    // fuzz_runs = 10000
}
```

Fuzzing configuration:

```toml
[profile.default]
fuzz_runs = 256              # Number of fuzz runs
fuzz_max_test_rejects = 65536  # Max rejected inputs

[profile.deep]
fuzz_runs = 10000  # More thorough fuzzing
```

## Invariant Testing

Invariant tests check properties that should always hold:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Token is ERC20 {
    constructor() ERC20("Test", "TST") {
        _mint(msg.sender, 1000000e18);
    }
}

contract TokenHandler is Test {
    Token public token;
    address[] public actors;
    
    constructor(Token _token) {
        token = _token;
    }
    
    function transfer(uint256 actorIndexSeed, uint256 amount) public {
        address from = actors[bound(actorIndexSeed, 0, actors.length - 1)];
        address to = address(uint160(bound(uint256(keccak256(abi.encode(amount))), 1, type(uint160).max)));
        
        vm.prank(from);
        try token.transfer(to, amount) {} catch {}
    }
    
    function addActor(address actor) public {
        actors.push(actor);
    }
}

contract TokenInvariantTest is Test {
    Token token;
    TokenHandler handler;
    
    function setUp() public {
        token = new Token();
        handler = new TokenHandler(token);
        
        // Set up actors
        address[] memory actors = new address[](3);
        actors[0] = address(0x1);
        actors[1] = address(0x2);
        actors[2] = address(0x3);
        
        for (uint i = 0; i < actors.length; i++) {
            handler.addActor(actors[i]);
            deal(address(token), actors[i], 100000e18);
        }
        
        // Target handler for invariant testing
        targetContract(address(handler));
    }
    
    // Invariant: Total supply should never change
    function invariant_TotalSupply() public {
        assertEq(token.totalSupply(), 1000000e18 + 300000e18);
    }
    
    // Invariant: Sum of balances equals total supply
    function invariant_BalanceSum() public {
        uint256 sum = 0;
        for (uint i = 0; i < 3; i++) {
            sum += token.balanceOf(handler.actors(i));
        }
        sum += token.balanceOf(address(this));
        assertEq(sum, token.totalSupply());
    }
}
```

Invariant configuration:

```toml
[invariant]
runs = 256
depth = 15
fail_on_revert = false
call_override = false
```

## Fork Testing

Test against live Base mainnet state:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IAerodromeRouter {
    struct Route {
        address from;
        address to;
        bool stable;
        address factory;
    }
    
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        Route[] calldata routes,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}

contract AerodromeBaseTest is Test {
    IERC20 constant USDC = IERC20(0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913);
    IERC20 constant WETH = IERC20(0x4200000000000000000000000000000000000006);
    IAerodromeRouter constant ROUTER = IAerodromeRouter(0xcF77a3Ba9A5CA399B7c97c74d54e5b1Beb874E43);
    
    address constant USDC_WHALE = 0xcdac0d6c6c59727a65f871236188350531885c43;
    address alice = address(0x1);

    function setUp() public {
        // Fork Base mainnet
        string memory rpcUrl = vm.envString("BASE_RPC_URL");
        vm.createSelectFork(rpcUrl);
        
        // Or fork at specific block
        // vm.createSelectFork(rpcUrl, 10000000);
    }

    function testSwapOnAerodrome() public {
        uint256 swapAmount = 1000e6;  // 1000 USDC
        
        // Fund alice with USDC from whale
        vm.prank(USDC_WHALE);
        USDC.transfer(alice, swapAmount);
        
        vm.startPrank(alice);
        
        // Approve router
        USDC.approve(address(ROUTER), swapAmount);
        
        // Prepare swap route
        IAerodromeRouter.Route[] memory routes = new IAerodromeRouter.Route[](1);
        routes[0] = IAerodromeRouter.Route({
            from: address(USDC),
            to: address(WETH),
            stable: false,
            factory: address(0x420DD381b31aEf6683db6B902084cB0FFECe40Da)
        });
        
        // Execute swap
        uint256 wethBefore = WETH.balanceOf(alice);
        ROUTER.swapExactTokensForTokens(
            swapAmount,
            0,  // Accept any amount
            routes,
            alice,
            block.timestamp + 3600
        );
        uint256 wethAfter = WETH.balanceOf(alice);
        
        vm.stopPrank();
        
        // Verify swap succeeded
        assertGt(wethAfter, wethBefore);
        assertEq(USDC.balanceOf(alice), 0);
    }
    
    function testForkMoonwellSupply() public {
        // Test Moonwell USDC market interaction
        address moonwellUSDC = 0xEdc817A28E8B93B03976FBd4a3dDBc9f7D176c22;
        
        vm.prank(USDC_WHALE);
        USDC.transfer(alice, 10000e6);
        
        vm.startPrank(alice);
        USDC.approve(moonwellUSDC, 10000e6);
        
        // Supply USDC (simplified - actual Moonwell interface may differ)
        (bool success,) = moonwellUSDC.call(
            abi.encodeWithSignature("mint(uint256)", 10000e6)
        );
        assertTrue(success);
        
        vm.stopPrank();
    }
}
```

Fork testing commands:

```bash
# Test with fork
forge test --fork-url https://mainnet.base.org

# Test with fork at specific block
forge test --fork-url https://mainnet.base.org --fork-block-number 10000000

# Cache fork data (faster subsequent runs)
forge test --fork-url https://mainnet.base.org --cache-path ~/.foundry/cache/base
```

## Gas Profiling

Analyze gas usage:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract GasTest is Test {
    function testGasUsage() public {
        uint256 gasBefore = gasleft();
        
        // Operation to measure
        uint256 result = 0;
        for (uint i = 0; i < 100; i++) {
            result += i;
        }
        
        uint256 gasUsed = gasBefore - gasleft();
        console.log("Gas used:", gasUsed);
    }
}
```

Run with gas reporting:

```bash
forge test --gas-report

# Report for specific contracts
forge test --gas-report --match-contract MyContract
```

## Common Patterns

**Pattern 1: Test Fixtures**

```solidity
contract TestFixtures is Test {
    function _deployToken() internal returns (ERC20) {
        return new ERC20("Test", "TST");
    }
    
    function _fundAccount(address account, uint256 amount) internal {
        vm.deal(account, amount);
    }
}

contract MyTest is TestFixtures {
    function testWithFixtures() public {
        ERC20 token = _deployToken();
        _fundAccount(address(0x1), 100 ether);
    }
}
```

**Pattern 2: Setup Multiple Scenarios**

```solidity
function _setupScenario1() internal {
    // Setup code
}

function _setupScenario2() internal {
    // Different setup
}

function testScenario1() public {
    _setupScenario1();
    // Test
}
```

## Gotchas

**Gotcha 1: setUp runs before EACH test**

```solidity
// This runs before testA and testB separately
function setUp() public {
    counter = new Counter();  // Fresh instance each time
}
```

**Gotcha 2: vm.assume vs bound**

Use `bound` instead of `vm.assume` for ranges to avoid rejections:

```solidity
// Prefer this
amount = bound(amount, 1, 1000);

// Over this
vm.assume(amount >= 1 && amount <= 1000);
```

**Gotcha 3: Fork persistence**

State changes persist during a fork test:

```solidity
function testFork() public {
    uint256 balanceBefore = USDC.balanceOf(alice);
    deal(address(USDC), alice, 1000e6);  // This modifies fork state
    // Subsequent operations see modified state
}
```

**Gotcha 4: Block context in forks**

Forked block has real timestamp/number:

```solidity
vm.createSelectFork("https://mainnet.base.org");
console.log(block.number);  // Real Base block number
vm.warp(1704067200);  // Override for testing
```