# Mathematical Precision Audit Report

**Contract:** [Contract Name]
**Files Analyzed:** [List of .sol files]
**Vulnerabilities Found:** Critical: X | High: Y | Medium: Z | Low: W

---

## [SEVERITY] Vulnerability Title
**Pattern:** #[1-13]
**File:** `path/to/Contract.sol`
**Lines:** [line numbers]
**Function:** `functionName()`

### Description
[2-3 sentences explaining the issue and why it matters]

### Vulnerable Code
```solidity
// Actual code from contract with line numbers
145: function calculateRewards(uint256 amount) public {
146:     uint256 reward = (amount / PRECISION) * rewardRate; // Division before multiplication
147:     userRewards[msg.sender] += reward;
148: }
```

### Impact
- **Loss Magnitude:** Up to [X]% precision loss per transaction
- **Exploitability:** [High/Medium/Low] - [Explanation of preconditions]
- **Affected Functions:** [List if multiple functions share this pattern]

### Proof of Concept
```solidity
function testExploit() public {
    // Setup
    uint256 smallAmount = 999; // Below PRECISION threshold

    // Execute vulnerable function
    contract.calculateRewards(smallAmount);

    // Demonstrate loss
    // Expected: X, Actual: 0 (rounded down)
    assert(userRewards[msg.sender] == 0); // User receives nothing
}
```

### Remediation
```solidity
// Fixed code
function calculateRewards(uint256 amount) public {
    uint256 reward = (amount * rewardRate) / PRECISION; // Multiply first
    require(reward > 0, "Amount too small");
    userRewards[msg.sender] += reward;
}
```

**Gas Impact:** +[X] gas per call (negligible vs security improvement)

---

## Summary

### Critical Issues Requiring Immediate Attention
1. [Issue #X] - [Brief description] - [Estimated loss potential]
2. [Issue #Y] - [Brief description] - [Estimated loss potential]

### Recommendations
- Standardize scaling: Use consistent WAD (1e18) or RAY (1e27) throughout
- Add minimum amount thresholds for all fee/reward calculations
- Implement SafeCast for all downcast operations
- Document rounding direction policy in natspec comments
- Query token decimals dynamically; avoid hardcoded assumptions
