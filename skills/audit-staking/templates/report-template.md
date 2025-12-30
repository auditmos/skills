# Staking & Reward Security Audit Report

## Executive Summary

**Contract:** [Contract Name]
**Audit Date:** [Date]
**Auditor:** [Name/Team]

**Findings Overview:**
- Critical: X
- High: X
- Medium: X
- Low: X

## Findings

---

### [SEVERITY] Finding #X: [Vulnerability Title]

**Pattern:** #X - [Pattern Name from reference.md]

**Location:** `[contract_name.sol:line_numbers]`

**Description:**

[Detailed explanation of the staking/reward vulnerability]

**Vulnerable Code:**

```solidity
function deposit(uint256 amount) external {
    // Missing: minimum stake, time lock, update call, etc.
    totalSupply += amount;
}
```

**Reward Theft/Dilution Analysis:**

[Analyze exploitation scenario:]
- **Attack vector:** [First deposit, direct transfer, flash action, etc.]
- **Rewards at risk:** [Amount of rewards attacker can steal/dilute]
- **Impact on users:** [% of rewards lost by legitimate stakers]
- **Attack cost:** [Gas, capital, etc.]

**Proof of Concept:**

```solidity
contract StakingExploit {
    VulnerableStaking target;

    function attack() external {
        // 1. Setup: Initial protocol state
        // 2. Exploit: Execute attack (front-run, dilute, etc.)
        // 3. Profit: Extract rewards or grief stakers
    }
}
```

**Attack Flow:**
1. [Initial state - total staked, rewards pending]
2. [Attacker action - deposit, transfer, etc.]
3. [State change - how totalSupply/rewards affected]
4. [Profit extraction or grief impact]
5. [Final state - user losses quantified]

**Impact Analysis:**

**Direct Impact:**
- [Immediate reward theft - e.g., "Attacker steals $X of initial rewards"]
- [User loss - e.g., "Legitimate stakers lose Y% of rewards"]

**Systemic Impact:**
- [Repeated exploitation - e.g., "Every new reward distribution vulnerable"]
- [Economic damage - e.g., "Protocol reward distribution broken"]

**Affected Users:**
- [Quantify impact - e.g., "All stakers during reward period"]

**Remediation:**

```solidity
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
```

**Recommendations:**
1. [Primary fix - e.g., "Use different tokens for staking and rewards"]
2. [Secondary fix - e.g., "Implement minimum stake of 1000 tokens"]
3. [Defense in depth - e.g., "Add 1-day time lock on withdrawals"]

**Gas Impact:** [Estimated additional gas cost]
- Time lock tracking: ~5,000 gas
- Minimum stake check: ~100 gas

---

### [SEVERITY] Finding #X: [Next Vulnerability]

[Repeat above structure for each finding]

---

## Severity Definitions

**Critical:** First depositor can steal all initial rewards, direct transfer dilution enabling reward theft, flash deposit/withdraw draining rewards.

**High:** Precision loss causing rewards to round to zero for legitimate users, stale index after distribution causing incorrect calculations.

**Medium:** Suboptimal reward timing reducing efficiency, griefing via flash actions without direct theft.

**Low:** Gas inefficiencies in calculations, missing events for transparency.

## Recommendations Summary

### Immediate Actions (Critical/High)
1. [List critical fixes]
   - Example: "Change reward token from WETH to different token (USDC)"
   - Example: "Track totalSupply separately from token balance"
   - Example: "Implement minimum stake of 1000 tokens"

### Short-term Improvements (Medium)
1. [List medium-priority enhancements]
   - Example: "Add 1-day time lock on withdrawals"
   - Example: "Call updateReward() before notifyRewardAmount()"

### Long-term Enhancements (Low)
1. [List optimization opportunities]
   - Example: "Optimize gas usage in reward calculations"

## Checklist Results

Based on `checklist.md`:

- [x] **Separate tokens:** Reward token differs from staking token ✓/✗
- [x] **No direct transfer dilution:** totalSupply tracked separately ✓/✗
- [x] **Precision protection:** Minimum stake or scaling implemented ✓/✗
- [x] **Flash protection:** Time locks or minimum duration ✓/✗
- [x] **Index updates:** updateReward called properly ✓/✗
- [x] **Balance integrity:** Cached balances correct during claims ✓/✗

## Token Configuration

- **Staking token:** [Address and symbol]
- **Reward token:** [Address and symbol]
- **Same token?** [Yes ❌ / No ✓]

## Staking Parameters

- **Minimum stake:** [Amount or "None" ❌]
- **Lock duration:** [Duration or "None" ❌]
- **Reward rate:** [Amount per second]
- **Total rewards:** [Amount available]

## Reward Distribution Analysis

### First Deposit Scenario

```
Initial rewards: 1000 tokens
First depositor: 1 wei
Attack profit: 990+ tokens (99%+)
Legitimate users: <10 tokens (<1%)
```

### Direct Transfer Dilution

```
Legitimate stakers: 10,000 tokens staked
Attacker sends: 10,000 tokens directly
Total supply: 20,000 (50% dilution)
Reward loss: 50% for all stakers
```

### Precision Loss Calculation

```
Minimum stake: 10 wei
Total supply: 1,000,000 tokens
Reward per token: 1e18 / 1,000,000e18 = 1e-6
User reward: 10 * 1e-6 / 1e18 = 0 (rounds to zero)
```

### Flash Attack Economics

```
Attacker capital: 1,000,000 tokens
Legitimate stakers: 100,000 tokens total
Attack in single block:
  1. Deposit 1M (total = 1.1M)
  2. Dilute rewards by 91%
  3. Withdraw 1M
  4. Net cost: gas only (~$50)
  5. User loss: 91% of 1 block rewards
  6. Repeatable every block
```

## Testing Recommendations

### Unit Tests
- [ ] First depositor attack prevention
- [ ] Direct transfer dilution protection
- [ ] Precision loss with small stakes
- [ ] Flash deposit/withdraw griefing
- [ ] Update call timing validation
- [ ] Balance caching correctness

### Integration Tests
- [ ] Multi-user reward distribution
- [ ] Reward notification with pending stakes
- [ ] Time lock enforcement
- [ ] Minimum stake validation across operations

### Scenario Tests
- [ ] Initial reward distribution to first stakers
- [ ] High-frequency deposit/withdraw patterns
- [ ] Extreme stake sizes (dust and whale)
- [ ] Reward distribution edge cases (zero stakers, etc.)

## Appendix

### Synthetix-Style Reward Distribution

```solidity
// Standard pattern used by Synthetix and forks
modifier updateReward(address account) {
    rewardPerTokenStored = rewardPerToken();
    lastUpdateTime = lastTimeRewardApplicable();

    if (account != address(0)) {
        rewards[account] = earned(account);
        userRewardPerTokenPaid[account] = rewardPerTokenStored;
    }
    _;
}

function rewardPerToken() public view returns (uint256) {
    if (totalSupply == 0) {
        return rewardPerTokenStored;
    }
    return rewardPerTokenStored + (
        (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply
    );
}

function earned(address account) public view returns (uint256) {
    return balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
        + rewards[account];
}
```

### Minimum Stake Calculation

```
Target: Ensure rewards > 1 token per distribution period

Reward rate: R tokens per second
Distribution period: D seconds
Total rewards: R * D
Minimum share: 0.01% (to get >1 token)

Required stake: (R * D / 0.0001) = minimum

Example:
- 1000 tokens over 7 days
- R = 1000 / 604800 = 0.00165 tokens/sec
- Target: >1 token per week
- Min share: 1/1000 = 0.1%
- Min stake: totalSupply * 0.001
- If totalSupply = 1M, min = 1000 tokens
```

### Time Lock Patterns

```solidity
// Pattern 1: Fixed lock from deposit
mapping(address => uint256) public depositTime;

function withdraw() external {
    require(block.timestamp >= depositTime[msg.sender] + LOCK, "Locked");
}

// Pattern 2: Weighted average lock
mapping(address => uint256) public weightedLockTime;

function deposit(uint256 amount) external {
    weightedLockTime[msg.sender] =
        (weightedLockTime[msg.sender] * balances[msg.sender] + block.timestamp * amount)
        / (balances[msg.sender] + amount);
}
```

### Direct Transfer Protection

```solidity
// WRONG: Uses balance
function rewardPerToken() public view returns (uint256) {
    uint256 totalSupply = stakingToken.balanceOf(address(this));
    // Attacker can inflate via direct transfer
}

// CORRECT: Uses tracked supply
uint256 public totalSupply; // State variable

function stake(uint256 amount) external {
    totalSupply += amount; // Only increments through stake
}

function rewardPerToken() public view returns (uint256) {
    // Uses tracked value, immune to direct transfers
    if (totalSupply == 0) return rewardPerTokenStored;
}
```

### Precision Considerations

```
Token decimals: 18
Reward decimals: 18
Scaling factor: 1e18

Calculation:
earned = balance * (rewardPerToken - userPaid) / 1e18

Minimum balance for non-zero rewards:
balance * rewardPerToken / 1e18 >= 1
balance >= 1e18 / rewardPerToken

If rewardPerToken = 1e12 (small rewards):
  Min balance = 1e18 / 1e12 = 1e6 wei = 0.000001 tokens

With MIN_STAKE = 1000e18:
  Always earned >= 1000e18 * rewardPerToken / 1e18
  = 1000 * rewardPerToken
  = significant amount
```
