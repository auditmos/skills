# Slippage Protection Audit Report

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
[2-3 sentences explaining the issue and what attack it enables]

### Vulnerable Code
```solidity
// Actual code from contract with line numbers
145: function swap(uint256 amountIn) external {
146:     router.swapExactTokensForTokens(
147:         amountIn,
148:         0, // No slippage protection
149:         path,
150:         msg.sender,
151:         block.timestamp // No deadline protection
152:     );
153: }
```

### Impact
- **MEV Extraction:** Up to [X]% of swap value extractable via sandwich attack
- **Exploitability:** [High/Medium/Low] - [Explanation of preconditions]
- **Daily Volume at Risk:** $[X] based on [source]
- **Affected Functions:** [List if multiple functions share this pattern]

### Proof of Concept
```solidity
// Sandwich attack simulation
function testSandwichAttack() public {
    // 1. Attacker front-runs with large buy
    vm.prank(attacker);
    uint256 attackBuyAmount = 1000 ether;
    router.swapExactTokensForTokens(attackBuyAmount, 0, path, attacker, block.timestamp);

    // 2. Victim transaction executes at inflated price (no slippage protection)
    vm.prank(victim);
    uint256 victimAmount = 10 ether;
    uint256 victimOut = router.swapExactTokensForTokens(victimAmount, 0, path, victim, block.timestamp);

    // 3. Attacker back-runs with sell
    vm.prank(attacker);
    router.swapExactTokensForTokens(attackBuyAmount, 0, reversePath, attacker, block.timestamp);

    // Demonstrate profit
    uint256 attackerProfit = attacker.balance - initialBalance;
    uint256 victimLoss = expectedOut - victimOut;

    // Attacker extracts ~40% of victim's intended output
    assertGt(attackerProfit, victimLoss * 35 / 100);
}
```

### Remediation
```solidity
// Fixed code
function swap(uint256 amountIn, uint256 minAmountOut, uint256 deadline) external {
    require(block.timestamp <= deadline, "Transaction expired");
    require(minAmountOut > 0, "Invalid slippage");

    router.swapExactTokensForTokens(
        amountIn,
        minAmountOut,
        path,
        msg.sender,
        deadline
    );
}
```

**Gas Impact:** +[X] gas per call (negligible vs security improvement)

---

## Summary

### Critical Issues Requiring Immediate Attention
1. [Issue #X] - [Brief description] - [Estimated MEV extraction potential]
2. [Issue #Y] - [Brief description] - [Estimated MEV extraction potential]

### Recommendations
- Accept user-provided slippage and deadline on all swap functions
- Calculate minAmountOut off-chain via front-end or keeper services
- Normalize all token amounts to common decimal base before slippage calculation
- For Uniswap V3, query multiple fee tiers and route through optimal pool
- Document recommended slippage values per asset pair (0.5% stables, 2% volatile)
- Implement emergency withdrawal mechanisms with relaxed slippage for black swan events
- Use TWAP oracles for on-chain price validation, never current block reserves
- Add circuit breakers that pause swaps during >50% single-block price moves
