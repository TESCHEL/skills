# Solidity Fundamentals

## Overview

This lesson covers core Solidity language features for version 0.8.x and above, which is the recommended version for Base deployments. Solidity 0.8.x includes built-in overflow/underflow protection, improved error handling with custom errors, and enhanced type safety.

## Version Pragma and License

Always specify the Solidity version and SPDX license identifier:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyContract {
    // Contract code
}
```

Use `^0.8.20` to allow compatible compiler versions. For production contracts, consider locking to a specific version like `0.8.20` to ensure consistent compilation.

## Value Types

Solidity provides several value types:

```solidity
contract ValueTypes {
    // Boolean
    bool public isActive = true;
    
    // Unsigned integers (0 to 2^n - 1)
    uint8 public smallNumber;   // 0 to 255
    uint256 public largeNumber; // 0 to 2^256 - 1
    uint public defaultUint;    // Alias for uint256
    
    // Signed integers (-2^(n-1) to 2^(n-1) - 1)
    int8 public smallSigned;    // -128 to 127
    int256 public largeSigned;  // -2^255 to 2^255 - 1
    
    // Address types
    address public owner;
    address payable public recipient;
    
    // Bytes (fixed-size)
    bytes1 public singleByte;
    bytes32 public hash;
    
    // Enums
    enum Status { Pending, Active, Completed, Cancelled }
    Status public currentStatus;
    
    constructor() {
        owner = msg.sender;
        currentStatus = Status.Pending;
    }
}
```

## Reference Types and Data Location

Reference types must specify a data location: `storage`, `memory`, or `calldata`.

```solidity
contract ReferenceTypes {
    // Storage: persistent state variable
    uint[] public storageArray;
    mapping(address => uint) public balances;
    
    struct User {
        string name;
        uint256 balance;
        bool isActive;
    }
    
    mapping(address => User) public users;
    
    // Memory: temporary, function-scoped
    function processData(uint[] memory tempArray) public pure returns (uint) {
        uint sum = 0;
        for (uint i = 0; i < tempArray.length; i++) {
            sum += tempArray[i];
        }
        return sum;
    }
    
    // Calldata: read-only, cheapest for external function parameters
    function processCalldata(uint[] calldata data) external pure returns (uint) {
        uint sum = 0;
        for (uint i = 0; i < data.length; i++) {
            sum += data[i];
        }
        return sum;
    }
    
    // String operations
    function updateUser(address userAddr, string calldata newName) external {
        User storage user = users[userAddr];
        user.name = newName;
        user.isActive = true;
    }
}
```

## Visibility Modifiers

Four visibility levels control function and state variable access:

```solidity
contract Visibility {
    // State variables: public, internal, or private (no external)
    uint public publicVar = 100;        // Auto-generates getter
    uint internal internalVar = 200;    // Accessible in derived contracts
    uint private privateVar = 300;      // Only in this contract
    
    // Functions: public, external, internal, or private
    function publicFunc() public pure returns (string memory) {
        return "Called internally or externally";
    }
    
    function externalFunc() external pure returns (string memory) {
        return "Only callable externally";
    }
    
    function internalFunc() internal pure returns (string memory) {
        return "Callable in this contract and derived contracts";
    }
    
    function privateFunc() private pure returns (string memory) {
        return "Only callable in this contract";
    }
    
    function demonstrateCalls() public view returns (uint) {
        // Can call public and internal functions internally
        publicFunc();
        internalFunc();
        privateFunc();
        // Cannot call externalFunc() directly, must use this.externalFunc()
        return publicVar + internalVar + privateVar;
    }
}
```

## Events

Events enable efficient logging and off-chain monitoring:

```solidity
contract EventExample {
    // Events with indexed parameters (up to 3 indexed params)
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event StatusChanged(address indexed user, string oldStatus, string newStatus);
    
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        emit Transfer(msg.sender, to, amount);
    }
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
        emit Transfer(address(0), msg.sender, msg.value);
    }
}
```

## Custom Errors

Custom errors (Solidity 0.8.4+) are more gas-efficient than revert strings:

```solidity
contract CustomErrors {
    error InsufficientBalance(uint256 requested, uint256 available);
    error Unauthorized(address caller);
    error InvalidAddress(address addr);
    error TransferFailed();
    
    mapping(address => uint256) public balances;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert Unauthorized(msg.sender);
        }
        _;
    }
    
    function withdraw(uint256 amount) external {
        uint256 balance = balances[msg.sender];
        if (balance < amount) {
            revert InsufficientBalance(amount, balance);
        }
        
        balances[msg.sender] = balance - amount;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        if (!success) {
            revert TransferFailed();
        }
    }
    
    function setOwner(address newOwner) external onlyOwner {
        if (newOwner == address(0)) {
            revert InvalidAddress(newOwner);
        }
        owner = newOwner;
    }
}
```

## Inheritance

Solidity supports multiple inheritance with C3 linearization:

```solidity
contract Base {
    uint public baseValue;
    
    event BaseEvent(uint value);
    
    function setBaseValue(uint _value) public virtual {
        baseValue = _value;
        emit BaseEvent(_value);
    }
}

contract Middle is Base {
    uint public middleValue;
    
    function setBaseValue(uint _value) public virtual override {
        middleValue = _value * 2;
        super.setBaseValue(_value);
    }
}

contract Derived is Middle {
    uint public derivedValue;
    
    function setBaseValue(uint _value) public override {
        derivedValue = _value * 3;
        super.setBaseValue(_value);
    }
    
    function getAllValues() public view returns (uint, uint, uint) {
        return (baseValue, middleValue, derivedValue);
    }
}

abstract contract AbstractBase {
    function mustImplement() public virtual returns (uint);
    
    function implemented() public pure returns (string memory) {
        return "This is implemented";
    }
}

contract Implementation is AbstractBase {
    function mustImplement() public pure override returns (uint) {
        return 42;
    }
}
```

## Modifiers

Function modifiers enable reusable checks:

```solidity
contract ModifierExample {
    address public owner;
    bool public paused;
    mapping(address => bool) public whitelist;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    modifier onlyWhitelisted() {
        require(whitelist[msg.sender], "Not whitelisted");
        _;
    }
    
    function pause() external onlyOwner {
        paused = true;
    }
    
    function unpause() external onlyOwner {
        paused = false;
    }
    
    function addToWhitelist(address addr) external onlyOwner {
        whitelist[addr] = true;
    }
    
    function performAction() external whenNotPaused onlyWhitelisted {
        // Action logic here
    }
}
```

## Common Patterns

**Receive and Fallback Functions:**

```solidity
contract PaymentReceiver {
    event Received(address sender, uint amount);
    event FallbackCalled(address sender, uint amount, bytes data);
    
    // Receive function for plain ETH transfers
    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
    
    // Fallback function for calls with data or when receive() doesn't exist
    fallback() external payable {
        emit FallbackCalled(msg.sender, msg.value, msg.data);
    }
}
```

## Gotchas

- **Integer Division**: Solidity rounds down. `5 / 2` equals `2`, not `2.5`.
- **Uninitialized Storage Pointers**: Always initialize storage references or use memory/calldata.
- **Default Visibility**: Functions without visibility are `public` by default. Always specify explicitly.
- **Array Length**: Dynamic array `.length` is modifiable in storage but read-only in memory.
- **Mapping Defaults**: All mapping keys exist with default values (0, false, empty).
- **Block Timestamp**: Use `block.timestamp` (seconds since epoch), not `block.number`, for time-dependent logic on Base.
- **Gas Estimation**: Always test gas usage on Base testnet before mainnet deployment.