# Reentrancy Vulnerability Examples

## Pattern #1: Token Transfer Reentrancy

### VULNERABLE
```solidity
contract VulnerableERC777 {
    mapping(address => uint256) public balances;

    // ISSUE: ERC777 token can reenter during transfer
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        // ERC777 tokensReceived callback triggered here
        // Attacker can reenter withdraw() before balance updated
        token.transfer(msg.sender, amount);

        // State updated AFTER external call - classic reentrancy
        balances[msg.sender] -= amount;
    }
}
```

### FIXED
```solidity
contract FixedERC777 {
    mapping(address => uint256) public balances;
    bool private locked;

    modifier nonReentrant() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }

    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        // CEI Pattern: Update state BEFORE external call
        balances[msg.sender] -= amount;

        // Safe: state already updated, reentrancy blocked
        token.transfer(msg.sender, amount);
    }
}
```

## Pattern #2: State Update After External Call

### VULNERABLE
```solidity
contract VulnerableWithdraw {
    mapping(address => uint256) public balances;

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // ISSUE: ETH sent before state update
        // Attacker can reenter via receive() fallback
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        // State updated after external call
        balances[msg.sender] = 0;
    }

    // Attack contract:
    // receive() external payable {
    //     if (address(vulnerable).balance > 0) {
    //         vulnerable.withdraw(); // Reenter!
    //     }
    // }
}
```

### FIXED
```solidity
contract FixedWithdraw {
    mapping(address => uint256) public balances;

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // CEI Pattern: Update state FIRST
        balances[msg.sender] = 0;

        // Safe: balance already zeroed, reentry has no effect
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
    }
}
```

## Pattern #3: Cross-Function Reentrancy

### VULNERABLE
```solidity
contract VulnerableCrossFunctionReentrancy {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public locked;

    // ISSUE: withdraw() and transfer() share balances state
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");

        // External call BEFORE state update
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Failed");

        // Attacker reenters transfer() here
        balances[msg.sender] -= amount;
    }

    function transfer(address to, uint256 amount) external {
        // Uses stale balances[msg.sender] value
        require(balances[msg.sender] >= amount, "Insufficient");

        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

    // Attack: withdraw() → reenter transfer() → drain funds
}
```

### FIXED
```solidity
contract FixedCrossFunctionReentrancy {
    mapping(address => uint256) public balances;
    bool private locked;

    modifier nonReentrant() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }

    // Both functions protected by same nonReentrant guard
    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient");

        balances[msg.sender] -= amount; // CEI pattern

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Failed");
    }

    function transfer(address to, uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient");

        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

## Pattern #4: Read-Only Reentrancy

### VULNERABLE
```solidity
contract VulnerablePool {
    uint256 public totalAssets;
    uint256 public totalShares;

    function withdraw(uint256 shares) external {
        uint256 assets = (shares * totalAssets) / totalShares;

        // ISSUE: totalAssets reduced before external call
        totalAssets -= assets;
        totalShares -= shares;

        // During callback, getPrice() returns inflated value
        token.transfer(msg.sender, assets);
    }

    // View function reads stale state during reentrancy
    function getPrice() public view returns (uint256) {
        // If called during withdraw callback:
        // totalAssets reduced but totalShares not yet updated
        // Returns incorrect price
        return (totalAssets * 1e18) / totalShares;
    }
}

contract ExternalProtocol {
    VulnerablePool pool;

    function liquidate(address user) external {
        // ISSUE: Reads price during reentrancy
        uint256 price = pool.getPrice(); // Stale/manipulated value

        // Makes decision based on wrong price
        uint256 collateralValue = userShares * price / 1e18;
        // ... liquidation logic
    }
}
```

### FIXED
```solidity
contract FixedPool {
    uint256 public totalAssets;
    uint256 public totalShares;
    bool private locked;

    modifier nonReentrant() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }

    function withdraw(uint256 shares) external nonReentrant {
        uint256 assets = (shares * totalAssets) / totalShares;

        totalAssets -= assets;
        totalShares -= shares;

        token.transfer(msg.sender, assets);
    }

    function getPrice() public view returns (uint256) {
        // If locked, price calculation may be inconsistent
        return (totalAssets * 1e18) / totalShares;
    }

    // Alternative: Add reentrancy check to view function
    function getPriceSafe() public view returns (uint256) {
        require(!locked, "Price inconsistent during operation");
        return (totalAssets * 1e18) / totalShares;
    }
}
```

## Advanced Example: Cross-Contract Reentrancy

### VULNERABLE
```solidity
contract VaultA {
    mapping(address => uint256) public deposits;

    function withdraw() external {
        uint256 amount = deposits[msg.sender];

        // Calls VaultB during withdrawal
        IVaultB(vaultB).notifyWithdrawal(msg.sender, amount);

        // State updated after external call
        deposits[msg.sender] = 0;
        token.transfer(msg.sender, amount);
    }
}

contract VaultB {
    function notifyWithdrawal(address user, uint256 amount) external {
        // Attacker reenters VaultA.withdraw() from here
        // VaultA.deposits[user] still non-zero
    }
}
```

### FIXED
```solidity
contract VaultA {
    mapping(address => uint256) public deposits;
    bool private locked;

    modifier nonReentrant() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }

    function withdraw() external nonReentrant {
        uint256 amount = deposits[msg.sender];

        // CEI: Update state FIRST
        deposits[msg.sender] = 0;

        // External calls after state changes
        IVaultB(vaultB).notifyWithdrawal(msg.sender, amount);
        token.transfer(msg.sender, amount);
    }
}
```

## Complex Example: ERC721 Callback Reentrancy

### VULNERABLE
```solidity
contract VulnerableNFTMarketplace {
    mapping(uint256 => address) public nftOwner;
    mapping(address => uint256) public balances;

    function buyNFT(uint256 tokenId) external payable {
        address seller = nftOwner[tokenId];
        uint256 price = getPrice(tokenId);

        require(msg.value >= price, "Insufficient payment");

        // ISSUE: NFT transfer triggers onERC721Received callback
        // Attacker can reenter before state updates
        nft.safeTransferFrom(address(this), msg.sender, tokenId);

        // State updated AFTER external call
        nftOwner[tokenId] = msg.sender;
        balances[seller] += price;

        // Attacker can call buyNFT again with same tokenId
        // before ownership recorded
    }
}
```

### FIXED
```solidity
contract FixedNFTMarketplace {
    mapping(uint256 => address) public nftOwner;
    mapping(address => uint256) public balances;

    function buyNFT(uint256 tokenId) external payable nonReentrant {
        address seller = nftOwner[tokenId];
        uint256 price = getPrice(tokenId);

        require(msg.value >= price, "Insufficient payment");

        // CEI: Update state FIRST
        nftOwner[tokenId] = msg.sender;
        balances[seller] += price;

        // Safe: state already updated
        nft.safeTransferFrom(address(this), msg.sender, tokenId);
    }
}
```

## Summary: Key Protections

1. **CEI Pattern:** Checks → Effects (state changes) → Interactions (external calls)
2. **NonReentrant modifier:** Use OpenZeppelin's ReentrancyGuard on all state-changing functions
3. **Token awareness:** Assume all tokens can have callbacks (ERC777, ERC721, malicious ERC20)
4. **Cross-function protection:** Apply nonReentrant to all functions sharing state
5. **Read-only safety:** Document view function limitations during reentrancy or add guards

## Attack Flow Template

```
1. User calls vulnerable function
2. Function performs checks
3. Function makes external call (transfer, call, etc.)
4. Attacker's callback triggered
5. Attacker reenters same or different function
6. Shared state still in old value
7. Exploit stale state to drain funds
8. Original function continues, updates state too late
```

## Detection Checklist

For each function with external calls:
- [ ] Is state updated before the external call?
- [ ] Is nonReentrant modifier present?
- [ ] Can other functions be reentered that share state?
- [ ] Do view functions return stale values during callback?
- [ ] Are token callbacks (ERC777, ERC721) considered?
