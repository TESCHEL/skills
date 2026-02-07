# Common Contract Patterns

## Overview

This lesson covers commonly used smart contract patterns and standards on Base, including token standards (ERC-20, ERC-721, ERC-1155), proxy patterns for upgradeability, and practical contract templates frequently deployed on Base.

## ERC-20 Token Standard

Fungible token implementation using OpenZeppelin:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, ERC20Burnable, ERC20Permit, Ownable {
    uint256 public constant MAX_SUPPLY = 1000000 * 10**18;
    
    event Minted(address indexed to, uint256 amount);
    
    error MaxSupplyExceeded(uint256 requested, uint256 available);
    
    constructor() 
        ERC20("MyToken", "MTK") 
        ERC20Permit("MyToken")
        Ownable(msg.sender) 
    {
        _mint(msg.sender, 100000 * 10**18);
    }
    
    function mint(address to, uint256 amount) external onlyOwner {
        if (totalSupply() + amount > MAX_SUPPLY) {
            revert MaxSupplyExceeded(totalSupply() + amount, MAX_SUPPLY);
        }
        _mint(to, amount);
        emit Minted(to, amount);
    }
    
    function decimals() public pure override returns (uint8) {
        return 18;
    }
}

contract ERC20WithSnapshot is ERC20 {
    struct Snapshot {
        uint256 id;
        uint256 value;
    }
    
    mapping(address => Snapshot[]) private accountSnapshots;
    mapping(uint256 => uint256) private totalSupplySnapshots;
    uint256 private currentSnapshotId;
    
    event SnapshotCreated(uint256 indexed id);
    
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}
    
    function snapshot() external returns (uint256) {
        currentSnapshotId++;
        emit SnapshotCreated(currentSnapshotId);
        return currentSnapshotId;
    }
    
    function balanceOfAt(address account, uint256 snapshotId) 
        public 
        view 
        returns (uint256) 
    {
        Snapshot[] storage snapshots = accountSnapshots[account];
        
        if (snapshots.length == 0) {
            return 0;
        }
        
        for (uint256 i = snapshots.length; i > 0; i--) {
            if (snapshots[i - 1].id <= snapshotId) {
                return snapshots[i - 1].value;
            }
        }
        
        return 0;
    }
    
    function _update(address from, address to, uint256 amount) internal virtual override {
        super._update(from, to, amount);
        
        if (from != address(0)) {
            updateSnapshot(accountSnapshots[from], balanceOf(from));
        }
        if (to != address(0)) {
            updateSnapshot(accountSnapshots[to], balanceOf(to));
        }
    }
    
    function updateSnapshot(Snapshot[] storage snapshots, uint256 currentValue) private {
        if (snapshots.length == 0 || snapshots[snapshots.length - 1].id < currentSnapshotId) {
            snapshots.push(Snapshot(currentSnapshotId, currentValue));
        } else {
            snapshots[snapshots.length - 1].value = currentValue;
        }
    }
}
```

## ERC-721 NFT Standard

Non-fungible token implementation:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract MyNFT is ERC721, ERC721URIStorage, ERC721Burnable, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private tokenIdCounter;
    
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public mintPrice = 0.01 ether;
    string public baseTokenURI;
    
    event Minted(address indexed to, uint256 indexed tokenId);
    event BaseURIUpdated(string newBaseURI);
    
    error MaxSupplyReached();
    error InsufficientPayment(uint256 sent, uint256 required);
    
    constructor() ERC721("MyNFT", "MNFT") Ownable(msg.sender) {
        baseTokenURI = "https://api.example.com/metadata/";
    }
    
    function mint(address to) external payable {
        if (tokenIdCounter.current() >= MAX_SUPPLY) {
            revert MaxSupplyReached();
        }
        
        if (msg.value < mintPrice) {
            revert InsufficientPayment(msg.value, mintPrice);
        }
        
        uint256 tokenId = tokenIdCounter.current();
        tokenIdCounter.increment();
        
        _safeMint(to, tokenId);
        emit Minted(to, tokenId);
    }
    
    function safeMint(address to, string memory uri) external onlyOwner {
        if (tokenIdCounter.current() >= MAX_SUPPLY) {
            revert MaxSupplyReached();
        }
        
        uint256 tokenId = tokenIdCounter.current();
        tokenIdCounter.increment();
        
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);
        emit Minted(to, tokenId);
    }
    
    function setBaseURI(string memory newBaseURI) external onlyOwner {
        baseTokenURI = newBaseURI;
        emit BaseURIUpdated(newBaseURI);
    }
    
    function setMintPrice(uint256 newPrice) external onlyOwner {
        mintPrice = newPrice;
    }
    
    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        (bool success, ) = owner().call{value: balance}("");
        require(success, "Withdrawal failed");
    }
    
    function _baseURI() internal view override returns (string memory) {
        return baseTokenURI;
    }
    
    function tokenURI(uint256 tokenId)
        public
        view
        override(ERC721, ERC721URIStorage)
        returns (string memory)
    {
        return super.tokenURI(tokenId);
    }
    
    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721URIStorage)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
    
    function _update(address to, uint256 tokenId, address auth)
        internal
        override(ERC721)
        returns (address)
    {
        return super._update(to, tokenId, auth);
    }
}
```

## ERC-1155 Multi-Token Standard

Multi-token standard supporting both fungible and non-fungible tokens:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";

contract MyERC1155 is ERC1155, Ownable, ERC1155Supply {
    string public name = "MyMultiToken";
    string public symbol = "MMT";
    
    mapping(uint256 => string) private tokenURIs;
    mapping(uint256 => uint256) public maxSupply;
    
    event TokenCreated(uint256 indexed tokenId, uint256 maxSupply, string uri);
    event TokenMinted(address indexed to, uint256 indexed tokenId, uint256 amount);
    
    error MaxSupplyExceeded(uint256 tokenId, uint256 requested, uint256 available);
    error InvalidTokenId(uint256 tokenId);
    
    constructor() ERC1155("https://api.example.com/metadata/{id}.json") Ownable(msg.sender) {}
    
    function createToken(
        uint256 tokenId,
        uint256 _maxSupply,
        string memory tokenURI
    ) external onlyOwner {
        require(maxSupply[tokenId] == 0, "Token already exists");
        
        maxSupply[tokenId] = _maxSupply;
        tokenURIs[tokenId] = tokenURI;
        
        emit TokenCreated(tokenId, _maxSupply, tokenURI);
    }
    
    function mint(
        address to,
        uint256 tokenId,
        uint256 amount,
        bytes memory data
    ) external onlyOwner {
        if (maxSupply[tokenId] == 0) {
            revert InvalidTokenId(tokenId);
        }
        
        if (totalSupply(tokenId) + amount > maxSupply[tokenId]) {
            revert MaxSupplyExceeded(
                tokenId,
                totalSupply(tokenId) + amount,
                maxSupply[tokenId]
            );
        }
        
        _mint(to, tokenId, amount, data);
        emit TokenMinted(to, tokenId, amount);
    }
    
    function mintBatch(
        address to,
        uint256[] memory tokenIds,
        uint256[] memory amounts,
        bytes memory data
    ) external onlyOwner {
        require(tokenIds.length == amounts.length, "Length mismatch");
        
        for (uint256 i = 0; i < tokenIds.length; i++) {
            if (maxSupply[tokenIds[i]] == 0) {
                revert InvalidTokenId(tokenIds[i]);
            }
            
            if (totalSupply(tokenIds[i]) + amounts[i] > maxSupply[tokenIds[i]]) {
                revert MaxSupplyExceeded(
                    tokenIds[i],
                    totalSupply(tokenIds[i]) + amounts[i],
                    maxSupply[tokenIds[i]]
                );
            }
        }
        
        _mintBatch(to, tokenIds, amounts, data);
    }
    
    function uri(uint256 tokenId) public view override returns (string memory) {
        if (bytes(tokenURIs[tokenId]).length > 0) {
            return tokenURIs[tokenId];
        }
        return super.uri(tokenId);
    }
    
    function _update(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values
    ) internal override(ERC1155, ERC1155Supply) {
        super._update(from, to, ids, values);
    }
}
```

## UUPS Proxy Pattern

Upgradeable contract using Universal Upgradeable Proxy Standard:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract MyUpgradeableContract is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;
    mapping(address => uint256) public balances;
    
    event ValueUpdated(uint256 oldValue, uint256 newValue);
    event Upgraded(address indexed implementation);
    
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }
    
    function initialize(address initialOwner) public initializer {
        __Ownable_init(initialOwner);
        __UUPSUpgradeable_init();
        value = 0;
    }
    
    function setValue(uint256 newValue) external onlyOwner {
        uint256 oldValue = value;
        value = newValue;
        emit ValueUpdated(oldValue, newValue);
    }
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
    
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {
        emit Upgraded(newImplementation);
    }
    
    function version() external pure returns (string memory) {
        return "1.0.0";
    }
}

contract MyUpgradeableContractV2 is MyUpgradeableContract {
    uint256 public newFeature;
    
    function setNewFeature(uint256 _newFeature) external onlyOwner {
        newFeature = _newFeature;
    }
    
    function version() external pure override returns (string memory) {
        return "2.0.0";
    }
}
```

## Transparent Proxy Pattern

Alternative proxy pattern with admin separation:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";

contract MyImplementation {
    uint256 public value;
    address public owner;
    bool private initialized;
    
    event ValueSet(uint256 newValue);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function initialize(address _owner) external {
        require(!initialized, "Already initialized");
        owner = _owner;
        initialized = true;
    }
    
    function setValue(uint256 _value) external onlyOwner {
        value = _value;
        emit ValueSet(_value);
    }
}

contract DeployTransparentProxy {
    event ProxyDeployed(address proxy, address admin, address implementation);
    
    function deploy() external returns (address proxy, address admin) {
        MyImplementation implementation = new MyImplementation();
        ProxyAdmin proxyAdmin = new ProxyAdmin(msg.sender);
        
        bytes memory initData = abi.encodeWithSelector(
            MyImplementation.initialize.selector,
            msg.sender
        );
        
        TransparentUpgradeableProxy transparentProxy = new TransparentUpgradeableProxy(
            address(implementation),
            address(proxyAdmin),
            initData
        );
        
        emit ProxyDeployed(address(transparentProxy), address(proxyAdmin), address(implementation));
        return (address(transparentProxy), address(proxyAdmin));
    }
}
```

## Timelock Controller

Governance pattern with time-delayed execution:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/governance/TimelockController.sol";

contract MyTimelock is TimelockController {
    constructor(
        uint256 minDelay,
        address[] memory proposers,
        address[] memory executors,
        address admin
    ) TimelockController(minDelay, proposers, executors, admin) {}
}

contract TimelockTarget {
    uint256 public value;
    address public timelock;
    
    event ValueChanged(uint256 newValue);
    
    error NotTimelock(address caller);
    
    constructor(address _timelock) {
        timelock = _timelock;
    }
    
    modifier onlyTimelock() {
        if (msg.sender != timelock) {
            revert NotTimelock(msg.sender);
        }
        _;
    }
    
    function setValue(uint256 newValue) external onlyTimelock {
        value = newValue;
        emit ValueChanged(newValue);
    }
}
```

## Common Patterns

**Factory Pattern:**

```solidity
contract TokenFactory {
    event TokenCreated(address indexed token, address indexed creator);
    
    function createToken(
        string memory name,
        string memory symbol
    ) external returns (address) {
        MyToken token = new MyToken();
        emit TokenCreated(address(token), msg.sender);
        return address(token);
    }
    
    function createTokenDeterministic(
        string memory name,
        string memory symbol,
        bytes32 salt
    ) external returns (address) {
        MyToken token = new MyToken{salt: salt}();
        emit TokenCreated(address(token), msg.sender);
        return address(token);
    }
}
```

**Registry Pattern:**

```solidity
contract Registry {
    mapping(bytes32 => address) public records;
    mapping(address => bool) public authorized;
    
    event RecordSet(bytes32 indexed key, address indexed value);
    
    modifier onlyAuthorized() {
        require(authorized[msg.sender], "Not authorized");
        _;
    }
    
    function setRecord(bytes32 key, address value) external onlyAuthorized {
        records[key] = value;
        emit RecordSet(key, value);
    }
    
    function getRecord(bytes32 key) external view returns (address) {
        return records[key];
    }
}
```

## Gotchas

- **Proxy Storage Collisions**: Maintain storage layout when upgrading contracts.
- **Constructor vs Initialize**: Upgradeable contracts use `initialize()` instead of constructors.
- **delegatecall in Proxies**: Proxies use delegatecall, which executes in proxy's storage context.
- **Token Approvals**: Always check and handle approval edge cases (0 approval, max approval).
- **ERC-721 Safe Transfers**: Use `safeTransferFrom` to prevent tokens being locked in contracts.
- **Metadata URIs**: Store URIs off-chain (IPFS) for cost efficiency.
- **Upgrade Authorization**: Carefully control who can upgrade proxy contracts.