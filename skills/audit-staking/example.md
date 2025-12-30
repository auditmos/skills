# Staking & Reward Vulnerability Examples

## Pattern #1: Front-Running First Deposit

### VULNERABLE
```solidity
contract VulnerableFirstDeposit {
    IERC20 public stakingToken; // WETH
    IERC20 public rewardToken; // WETH - SAME TOKEN!

    uint256 public totalSupply;
    uint256 public rewardPerTokenStored;
    mapping(address => uint256) public balances;

    // ISSUE: Reward token same as staking token
    function deposit(uint256 amount) external {
        totalSupply += amount;
        balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
    }

    function getReward() external {
        uint256 reward = earned(msg.sender);
        rewardToken.transfer(msg.sender, reward);
    }

    // Attack:
    // 1. Protocol deploys with 1000 WETH rewards
    // 2. Attacker front-runs first user's deposit(100 WETH)
    // 3. Attacker deposits 1 wei
    // 4. User's deposit executes
    // 5. Attacker withdraws 1 wei + claims 990 WETH rewards
    // 6. User only gets 10 WETH of original 1000 WETH
}
```

### FIXED
```solidity
contract FixedFirstDeposit {
    IERC20 public stakingToken; // WETH
    IERC20 public rewardToken; // USDC - DIFFERENT TOKEN!

    uint256 public totalSupply;
    mapping(address => uint256) public balances;

    // Reward token different from staking token
    // Initial rewards can't be stolen via first deposit

    function deposit(uint256 amount) external {
        require(amount >= MIN_DEPOSIT, "Amount too small");
        totalSupply += amount;
        balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
    }
}
```

## Pattern #2: Reward Dilution via Direct Transfer

### VULNERABLE
```solidity
contract VulnerableDilution {
    IERC20 public stakingToken;
    uint256 public rewardPerTokenStored;

    // ISSUE: Uses token balance for totalSupply
    function rewardPerToken() public view returns (uint256) {
        uint256 totalSupply = stakingToken.balanceOf(address(this));

        if (totalSupply == 0) return rewardPerTokenStored;

        // Attacker sends tokens directly to contract
        // totalSupply inflated without earning rights
        // Dilutes rewards for legitimate stakers

        return rewardPerTokenStored + (newRewards * 1e18 / totalSupply);
    }

    // Attack:
    // 1. 100 users stake 100 tokens each = 10k total
    // 2. 1000 rewards to distribute
    // 3. Attacker sends 10k tokens directly to contract
    // 4. totalSupply now 20k (10k staked + 10k direct)
    // 5. Rewards per token halved
    // 6. Legitimate stakers lose 50% of rewards
}
```

### FIXED
```solidity
contract FixedDilution {
    IERC20 public stakingToken;
    uint256 public totalSupply; // Tracked separately
    uint256 public rewardPerTokenStored;
    mapping(address => uint256) public balances;

    function stake(uint256 amount) external {
        totalSupply += amount; // Only increments through stake
        balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
    }

    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) return rewardPerTokenStored;

        // Uses tracked totalSupply, not balance
        // Direct transfers don't affect calculation
        return rewardPerTokenStored + (newRewards * 1e18 / totalSupply);
    }
}
```

## Pattern #3: Precision Loss in Reward Calculation

### VULNERABLE
```solidity
contract VulnerablePrecision {
    uint256 public rewardRate = 1e18; // 1 token per second
    uint256 public totalSupply;
    mapping(address => uint256) public balances;

    function earned(address account) public view returns (uint256) {
        uint256 duration = block.timestamp - lastUpdateTime;

        // ISSUE: Small stakes cause precision loss
        // Example: balance = 10 wei, totalSupply = 1000 ether
        // rewardPerToken = duration * 1e18 / 1000e18 = duration / 1000
        // earned = 10 * (duration / 1000) / 1e18 = 0 (rounds to zero!)

        uint256 rewardPerToken = duration * rewardRate / totalSupply;
        return balances[account] * rewardPerToken / 1e18;
    }

    // User with 10 wei stake earns 0 rewards forever
    // Accumulated loss significant over many users/time
}
```

### FIXED
```solidity
contract FixedPrecision {
    uint256 public rewardRate = 1e18;
    uint256 public totalSupply;
    uint256 public constant MIN_STAKE = 1000e18; // 1000 tokens minimum
    mapping(address => uint256) public balances;

    function stake(uint256 amount) external {
        require(
            balances[msg.sender] + amount >= MIN_STAKE,
            "Below minimum stake"
        );

        totalSupply += amount;
        balances[msg.sender] += amount;
    }

    function earned(address account) public view returns (uint256) {
        uint256 duration = block.timestamp - lastUpdateTime;

        // With minimum stake, precision loss avoided
        // Minimum 1000e18 * rewardPerToken / 1e18 always > 0

        uint256 rewardPerToken = duration * rewardRate / totalSupply;
        return balances[account] * rewardPerToken / 1e18;
    }
}
```

## Pattern #4: Flash Deposit/Withdraw Griefing

### VULNERABLE
```solidity
contract VulnerableFlash {
    uint256 public totalSupply;
    uint256 public rewardPerTokenStored;
    mapping(address => uint256) public balances;

    // ISSUE: No time lock or minimum duration
    function deposit(uint256 amount) external {
        updateReward(msg.sender);
        totalSupply += amount;
        balances[msg.sender] += amount;
        token.transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(uint256 amount) external {
        updateReward(msg.sender);
        totalSupply -= amount;
        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
    }

    // Attack (in one transaction):
    // Initial: 1000 users with 100 tokens each = 100k total
    // 1. Attacker deposits 1M tokens (totalSupply = 1.1M)
    // 2. updateReward() distributes pending rewards based on 1.1M
    // 3. Legitimate users get 100k/1.1M = 9% each instead of 100%
    // 4. Attacker immediately withdraws 1M tokens
    // 5. Attacker diluted 91% of reward distribution
    // 6. Repeat every block
}
```

### FIXED
```solidity
contract FixedFlash {
    uint256 public totalSupply;
    uint256 public constant LOCK_DURATION = 1 days;
    mapping(address => uint256) public balances;
    mapping(address => uint256) public depositTime;

    function deposit(uint256 amount) external {
        updateReward(msg.sender);
        totalSupply += amount;
        balances[msg.sender] += amount;
        depositTime[msg.sender] = block.timestamp;
        token.transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(uint256 amount) external {
        // Enforce minimum lock duration
        require(
            block.timestamp >= depositTime[msg.sender] + LOCK_DURATION,
            "Tokens locked"
        );

        updateReward(msg.sender);
        totalSupply -= amount;
        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
    }

    // Flash attacks prevented - must lock for 1 day
}
```

## Pattern #5: Update Not Called After Reward Distribution

### VULNERABLE
```solidity
contract VulnerableStaleIndex {
    uint256 public rewardPerTokenStored;
    uint256 public lastUpdateTime;
    uint256 public rewardRate;

    function updateReward(address account) internal {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        // ... update user rewards
    }

    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) return rewardPerTokenStored;

        uint256 duration = block.timestamp - lastUpdateTime;
        return rewardPerTokenStored + (duration * rewardRate / totalSupply);
    }

    // ISSUE: notifyRewardAmount doesn't call updateReward
    function notifyRewardAmount(uint256 reward) external {
        rewardRate = reward / DURATION;
        // Missing updateReward()!
        // rewardPerTokenStored not updated
        // Next calculation uses stale lastUpdateTime
    }

    // Result: Rewards double-counted or missed
}
```

### FIXED
```solidity
contract FixedStaleIndex {
    uint256 public rewardPerTokenStored;
    uint256 public lastUpdateTime;
    uint256 public rewardRate;

    function updateReward(address account) internal {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        // ... update user rewards
    }

    function notifyRewardAmount(uint256 reward) external {
        // Update index BEFORE changing rate
        updateReward(address(0));

        rewardRate = reward / DURATION;
        lastUpdateTime = block.timestamp;

        // Index synchronized with new rate
    }
}
```

## Pattern #6: Balance Caching Issues

### VULNERABLE
```solidity
contract VulnerableCache {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public rewards;
    uint256 public rewardPerTokenStored;

    function getReward() external {
        updateReward(msg.sender);

        uint256 reward = rewards[msg.sender];

        // ISSUE: Claiming updates balance incorrectly
        rewards[msg.sender] = 0;
        balances[msg.sender] += reward; // Wrong! Added to stake

        rewardToken.transfer(msg.sender, reward);

        // User's balance now inflated
        // Next earned() calculation uses wrong balance
        // Can claim more than earned
    }

    function earned(address account) public view returns (uint256) {
        // Uses inflated balance from previous claim
        return balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
            + rewards[account];
    }
}
```

### FIXED
```solidity
contract FixedCache {
    mapping(address => uint256) public balances; // Staked amount only
    mapping(address => uint256) public rewards; // Pending rewards
    uint256 public rewardPerTokenStored;

    function getReward() external {
        updateReward(msg.sender);

        uint256 reward = rewards[msg.sender];
        rewards[msg.sender] = 0;

        // Don't modify balances - only send reward
        rewardToken.transfer(msg.sender, reward);
    }

    function earned(address account) public view returns (uint256) {
        // Uses correct staked balance only
        return balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
            + rewards[account];
    }
}
```

## Complete Staking Contract Example

### VULNERABLE
```solidity
contract CompleteVulnerable {
    IERC20 public stakingToken; // WETH
    IERC20 public rewardToken; // WETH - same token!
    uint256 public rewardRate = 1e18;

    function deposit(uint256 amount) external {
        // No minimum, allows dust
        // Uses balance not tracked supply
        uint256 totalSupply = stakingToken.balanceOf(address(this));
        totalSupply += amount;

        stakingToken.transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(uint256 amount) external {
        // No time lock
        stakingToken.transfer(msg.sender, amount);
    }

    function notifyReward(uint256 amount) external {
        // Missing updateReward call
        rewardRate = amount / 7 days;
    }
}
```

### FIXED
```solidity
contract CompleteFixed {
    IERC20 public stakingToken; // WETH
    IERC20 public rewardToken; // USDC - different!

    uint256 public totalSupply; // Tracked separately
    uint256 public rewardRate;
    uint256 public rewardPerTokenStored;
    uint256 public lastUpdateTime;
    uint256 public constant MIN_STAKE = 1000e18;
    uint256 public constant LOCK_DURATION = 1 days;

    mapping(address => uint256) public balances;
    mapping(address => uint256) public depositTime;
    mapping(address => uint256) public rewards;
    mapping(address => uint256) public userRewardPerTokenPaid;

    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;

        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

    function deposit(uint256 amount) external updateReward(msg.sender) {
        require(
            balances[msg.sender] + amount >= MIN_STAKE,
            "Below minimum"
        );

        totalSupply += amount;
        balances[msg.sender] += amount;
        depositTime[msg.sender] = block.timestamp;

        stakingToken.transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(uint256 amount) external updateReward(msg.sender) {
        require(
            block.timestamp >= depositTime[msg.sender] + LOCK_DURATION,
            "Locked"
        );

        totalSupply -= amount;
        balances[msg.sender] -= amount;

        stakingToken.transfer(msg.sender, amount);
    }

    function getReward() external updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        rewards[msg.sender] = 0;
        rewardToken.transfer(msg.sender, reward);
    }

    function notifyRewardAmount(uint256 reward) external updateReward(address(0)) {
        rewardRate = reward / 7 days;
        lastUpdateTime = block.timestamp;
    }

    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) return rewardPerTokenStored;

        uint256 duration = block.timestamp - lastUpdateTime;
        return rewardPerTokenStored + (duration * rewardRate * 1e18 / totalSupply);
    }

    function earned(address account) public view returns (uint256) {
        return balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
            + rewards[account];
    }
}
```

## Summary: Key Protections

1. **Different tokens:** Reward token ≠ staking token
2. **Tracked supply:** totalSupply separate from balance
3. **Minimum stake:** Prevent precision loss (e.g., 1000 tokens)
4. **Time lock:** Minimum 1 day lock prevents flash attacks
5. **Update timing:** Call updateReward before reward distribution
6. **Balance integrity:** Don't modify stake balance during claims
