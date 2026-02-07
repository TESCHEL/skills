# Security Best Practices

## Overview

Smart contract security is critical because deployed contracts are immutable and often handle valuable assets. This lesson covers essential security patterns, common vulnerabilities, and mitigation strategies for Solidity development on Base.

## Reentrancy Protection

Reentrancy occurs when external calls allow malicious contracts to re-enter the calling function:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract VulnerableWithdrawal {
    mapping(address => uint256) public balances;
    
    // VULNERABLE: State updated after external call
    function vulnerableWithdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");
        
        // DANGER: External call before state update
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
        
        // Attacker can re-enter here before this executes
        balances[msg.sender] = 0;
    }
}

contract SecureWithdrawal is ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    error InsufficientBalance(uint256 requested, uint256 available);
    error TransferFailed();
    
    // SECURE: Using ReentrancyGuard
    function secureWithdrawGuard() external nonReentrant {
        uint256 balance = balances[msg.sender];
        if (balance == 0) {
            revert InsufficientBalance(1, 0);
        }
        
        balances[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: balance}("");
        if (!success) {
            revert TransferFailed();
        }
    }
    
    // SECURE: Checks-Effects-Interactions pattern
    function secureWithdrawPattern() external {
        // 1. CHECKS
        uint256 balance = balances[msg.sender];
        if (balance == 0) {
            revert InsufficientBalance(1, 0);
        }
        
        // 2. EFFECTS (state changes)
        balances[msg.sender] = 0;
        
        // 3. INTERACTIONS (external calls)
        (bool success, ) = msg.sender.call{value: balance}("");
        if (!success) {
            revert TransferFailed();
        }
    }
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
}
```

## Access Control

Implement robust access control using OpenZeppelin patterns:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

// Simple owner-based access control
contract SimpleAccessControl is Ownable {
    uint256 public value;
    
    constructor() Ownable(msg.sender) {}
    
    function setValue(uint256 newValue) external onlyOwner {
        value = newValue;
    }
    
    function transferOwnership(address newOwner) public override onlyOwner {
        require(newOwner != address(0), "Invalid address");
        super.transferOwnership(newOwner);
    }
}

// Role-based access control
contract RoleBasedAccessControl is AccessControl {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    
    mapping(address => uint256) public balances;
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ADMIN_ROLE, msg.sender);
    }
    
    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        balances[to] += amount;
    }
    
    function burn(address from, uint256 amount) external onlyRole(BURNER_ROLE) {
        require(balances[from] >= amount, "Insufficient balance");
        balances[from] -= amount;
    }
    
    function addMinter(address account) external onlyRole(ADMIN_ROLE) {
        grantRole(MINTER_ROLE, account);
    }
    
    function removeMinter(address account) external onlyRole(ADMIN_ROLE) {
        revokeRole(MINTER_ROLE, account);
    }
}

// Two-step ownership transfer
contract TwoStepOwnable {
    address public owner;
    address public pendingOwner;
    
    event OwnershipTransferStarted(address indexed previousOwner, address indexed newOwner);
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    error Unauthorized(address caller);
    error InvalidAddress(address addr);
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert Unauthorized(msg.sender);
        }
        _;
    }
    
    function transferOwnership(address newOwner) external onlyOwner {
        if (newOwner == address(0)) {
            revert InvalidAddress(newOwner);
        }
        pendingOwner = newOwner;
        emit OwnershipTransferStarted(owner, newOwner);
    }
    
    function acceptOwnership() external {
        if (msg.sender != pendingOwner) {
            revert Unauthorized(msg.sender);
        }
        address oldOwner = owner;
        owner = pendingOwner;
        pendingOwner = address(0);
        emit OwnershipTransferred(oldOwner, owner);
    }
}
```

## Integer Overflow Protection

Solidity 0.8.x includes automatic overflow/underflow checks:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SafeMath {
    // Solidity 0.8+ automatically reverts on overflow/underflow
    function safeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b; // Reverts on overflow
    }
    
    function safeSub(uint256 a, uint256 b) public pure returns (uint256) {
        return a - b; // Reverts on underflow
    }
    
    function safeMul(uint256 a, uint256 b) public pure returns (uint256) {
        return a * b; // Reverts on overflow
    }
    
    // Use unchecked for gas optimization when overflow is impossible or desired
    function uncheckedIncrement(uint256 value) public pure returns (uint256) {
        unchecked {
            return value + 1;
        }
    }
    
    // Example: Loop counter that won't overflow in practice
    function sumArray(uint256[] calldata values) public pure returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < values.length; ) {
            sum += values[i];
            unchecked {
                i++; // Safe: i < values.length, won't overflow
            }
        }
        return sum;
    }
}
```

## Checks-Effects-Interactions Pattern

Always order operations to prevent vulnerabilities:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ChecksEffectsInteractions {
    mapping(address => uint256) public balances;
    mapping(address => bool) public hasVoted;
    uint256 public totalVotes;
    
    error InsufficientBalance();
    error AlreadyVoted();
    error TransferFailed();
    
    // CORRECT: Checks-Effects-Interactions
    function withdraw(uint256 amount) external {
        // 1. CHECKS: Validate preconditions
        if (balances[msg.sender] < amount) {
            revert InsufficientBalance();
        }
        
        // 2. EFFECTS: Update state
        balances[msg.sender] -= amount;
        
        // 3. INTERACTIONS: External calls last
        (bool success, ) = msg.sender.call{value: amount}("");
        if (!success) {
            revert TransferFailed();
        }
    }
    
    function vote() external {
        // 1. CHECKS
        if (hasVoted[msg.sender]) {
            revert AlreadyVoted();
        }
        
        // 2. EFFECTS
        hasVoted[msg.sender] = true;
        totalVotes++;
        
        // 3. INTERACTIONS (if any)
        // External calls would go here
    }
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
}
```

## Input Validation

Always validate inputs to prevent unexpected behavior:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract InputValidation {
    uint256 public constant MAX_SUPPLY = 1000000 * 10**18;
    uint256 public constant MIN_DEPOSIT = 0.01 ether;
    
    mapping(address => uint256) public balances;
    
    error ZeroAddress();
    error InvalidAmount(uint256 amount);
    error ExceedsMaxSupply(uint256 requested, uint256 available);
    error BelowMinimum(uint256 amount, uint256 minimum);
    
    function transfer(address to, uint256 amount) external {
        // Validate address
        if (to == address(0)) {
            revert ZeroAddress();
        }
        
        // Validate amount
        if (amount == 0) {
            revert InvalidAmount(amount);
        }
        
        if (balances[msg.sender] < amount) {
            revert InvalidAmount(amount);
        }
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
    
    function mint(address to, uint256 amount) external {
        if (to == address(0)) {
            revert ZeroAddress();
        }
        
        uint256 totalSupply = getTotalSupply();
        if (totalSupply + amount > MAX_SUPPLY) {
            revert ExceedsMaxSupply(totalSupply + amount, MAX_SUPPLY);
        }
        
        balances[to] += amount;
    }
    
    function deposit() external payable {
        if (msg.value < MIN_DEPOSIT) {
            revert BelowMinimum(msg.value, MIN_DEPOSIT);
        }
        balances[msg.sender] += msg.value;
    }
    
    function getTotalSupply() public view returns (uint256) {
        // Implementation
        return 0;
    }
}
```

## Front-Running Protection

Mitigate front-running attacks with commit-reveal or other patterns:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract CommitReveal {
    struct Commitment {
        bytes32 commitment;
        uint256 timestamp;
        bool revealed;
    }
    
    mapping(address => Commitment) public commitments;
    uint256 public constant REVEAL_DELAY = 1 minutes;
    
    event Committed(address indexed user, bytes32 commitment);
    event Revealed(address indexed user, uint256 value);
    
    error AlreadyCommitted();
    error NotCommitted();
    error TooEarly(uint256 revealTime);
    error AlreadyRevealed();
    error InvalidReveal();
    
    // Step 1: Commit to a value hash
    function commit(bytes32 commitment) external {
        if (commitments[msg.sender].commitment != bytes32(0)) {
            revert AlreadyCommitted();
        }
        
        commitments[msg.sender] = Commitment({
            commitment: commitment,
            timestamp: block.timestamp,
            revealed: false
        });
        
        emit Committed(msg.sender, commitment);
    }
    
    // Step 2: Reveal the actual value
    function reveal(uint256 value, bytes32 salt) external {
        Commitment storage userCommit = commitments[msg.sender];
        
        if (userCommit.commitment == bytes32(0)) {
            revert NotCommitted();
        }
        
        if (block.timestamp < userCommit.timestamp + REVEAL_DELAY) {
            revert TooEarly(userCommit.timestamp + REVEAL_DELAY);
        }
        
        if (userCommit.revealed) {
            revert AlreadyRevealed();
        }
        
        bytes32 hash = keccak256(abi.encodePacked(msg.sender, value, salt));
        if (hash != userCommit.commitment) {
            revert InvalidReveal();
        }
        
        userCommit.revealed = true;
        emit Revealed(msg.sender, value);
        
        // Process the revealed value
    }
    
    // Helper to generate commitment off-chain
    function generateCommitment(address user, uint256 value, bytes32 salt) 
        public 
        pure 
        returns (bytes32) 
    {
        return keccak256(abi.encodePacked(user, value, salt));
    }
}
```

## Common Vulnerabilities

**Timestamp Dependence:**

```solidity
contract TimestampSafety {
    // AVOID: Exact timestamp comparison (miners can manipulate ~15 seconds)
    function badTimestamp() external view returns (bool) {
        return block.timestamp == 1234567890; // Never use ==
    }
    
    // GOOD: Use ranges or inequality
    function goodTimestamp(uint256 deadline) external view returns (bool) {
        return block.timestamp >= deadline;
    }
}
```

**Delegatecall Vulnerabilities:**

```solidity
contract DelegatecallSafety {
    address public owner;
    
    // DANGEROUS: Delegatecall to user-supplied address
    function dangerousDelegatecall(address target, bytes calldata data) external {
        (bool success, ) = target.delegatecall(data);
        require(success);
    }
    
    // SAFER: Whitelist allowed targets
    mapping(address => bool) public allowedTargets;
    
    function safeDelegatecall(address target, bytes calldata data) external {
        require(allowedTargets[target], "Target not allowed");
        (bool success, ) = target.delegatecall(data);
        require(success);
    }
}
```

## Common Patterns

**Pull Over Push Payments:**

```solidity
contract PullPayment {
    mapping(address => uint256) public pendingWithdrawals;
    
    function allowWithdrawal(address user, uint256 amount) internal {
        pendingWithdrawals[user] += amount;
    }
    
    function withdraw() external {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "No pending withdrawal");
        
        pendingWithdrawals[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

## Gotchas

- **tx.origin**: Never use `tx.origin` for authentication, use `msg.sender`.
- **Delegatecall Storage**: Delegatecall executes in caller's context, modifying caller's storage.
- **Selfdestruct**: Can force-send ETH to contracts, breaking balance-based logic.
- **Gas Limits**: External calls have 63/64 gas forwarded by default.
- **Private Data**: All contract data is publicly readable, even `private` variables.
- **Randomness**: Block properties are not truly random, use Chainlink VRF for randomness.