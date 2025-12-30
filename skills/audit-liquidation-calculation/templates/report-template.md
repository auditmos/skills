# Liquidation Calculation Security Audit Report

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

[Detailed explanation of the calculation vulnerability]

**Vulnerable Code:**

```solidity
// Highlight the problematic calculation
function liquidate(address user) external {
    // Show the specific calculation issue
}
```

**Economic Impact:**

[Analyze calculation correctness and profitability:]
- **Liquidator reward:** [Calculated value in token decimals]
- **Expected reward:** [What it should be]
- **Decimal error:** [If applicable - e.g., "1e18 assumed for 6-decimal token"]
- **Protocol fee:** [Amount and percentage of bonus]
- **Net liquidator profit:** [Reward - protocol_fee - gas_cost]
- **Is liquidation profitable?** [Yes/No with calculation]
- **Missing value:** [Yield/PNL not accounted for, if applicable]

**Proof of Concept:**

```solidity
// Executable test case showing calculation error or exploitation
contract PoC {
    function testIncorrectRewardCalculation() public {
        // 1. Setup position with specific decimals
        // 2. Trigger liquidation
        // 3. Calculate actual reward vs expected
        // 4. Demonstrate error

        // Expected: 100 USDC reward
        // Actual: 0.0001 USDC (decimal error)
    }
}
```

**Scenario:**
1. [Position setup - tokens, amounts, decimals]
2. [Position becomes liquidatable]
3. [Liquidator calls liquidate()]
4. [Calculation shows error - wrong decimals, unprioritized fee, missing yield]
5. [Result: unprofitable liquidation / unfair liquidation / value extraction]

**Impact Analysis:**

**Short-term:**
- [Immediate impact - e.g., "All liquidations fail due to 0 rewards"]

**Long-term:**
- [Systemic impact - e.g., "Bad debt accumulates, protocol insolvent"]

**Scale:**
- [Quantify affected liquidations - e.g., "All USDC collateral positions"]

**Remediation:**

```solidity
// Fixed implementation
function liquidate(address user) external {
    Position memory pos = positions[user];

    // Fix decimal handling
    uint256 debtScaled = pos.debt * (10 ** collateralDecimals)
        / (10 ** debtDecimals);

    // Calculate liquidator reward FIRST
    uint256 bonusAmount = (debtScaled * LIQUIDATION_BONUS) / 100;
    uint256 liquidatorReward = bonusAmount - debtScaled;

    // Protocol fee from reward
    uint256 protocolFee = (liquidatorReward * PROTOCOL_FEE_RATE) / 10000;
    uint256 netReward = liquidatorReward - protocolFee;

    // Include yield in collateral value
    uint256 collateral = vault.balanceOf(user); // Not userDeposits[user]

    require(collateral >= bonusAmount, "Insufficient collateral");

    // ... complete liquidation
}
```

**Recommendations:**
1. [Primary fix - e.g., "Scale rewards to collateral token decimals"]
2. [Fee structure - e.g., "Pay liquidator reward before protocol fees"]
3. [Valuation - e.g., "Include earned yield in collateral calculations"]
4. [Self-liquidation - e.g., "Add require(msg.sender != user)"]

**Gas Impact:** [Estimated additional gas cost for fix]

**Economic Calculation:**

Before fix:
```
Collateral: 1000 USDC (6 decimals)
Debt: 900 DAI (18 decimals)
Reward calc: (900e18 * 110) / 1e18 = 990e0 = 0.00000099 USDC
Net: unusable reward (liquidation impossible)
```

After fix:
```
Collateral: 1000 USDC (1000e6)
Debt scaled: 900 DAI → 900e6 (scaled to USDC decimals)
Bonus: (900e6 * 110) / 100 = 990e6
Reward: 990e6 - 900e6 = 90e6 = 90 USDC
Protocol fee: 90 * 5% = 4.5 USDC
Net reward: 85.5 USDC - $20 gas = $65.5 profit
```

---

### [SEVERITY] Finding #X: [Next Vulnerability]

[Repeat above structure for each finding]

---

## Severity Definitions

**Critical:** Incorrect liquidator reward decimals preventing all liquidations, profitable self-liquidation via oracle manipulation enabling value extraction. Protocol insolvency risk.

**High:** Unprioritized liquidator rewards removing liquidation incentive, excessive protocol fees making liquidation unprofitable, unaccounted yield causing unfair liquidations.

**Medium:** Minimum collateral not accounting for liquidation costs, missing swap fees during liquidation, liquidation unprofitable for small positions.

**Low:** Suboptimal fee structures, missing documentation, gas inefficiencies without security impact.

## Recommendations Summary

### Immediate Actions (Critical/High)
1. [List critical calculation fixes to deploy immediately]
   - Example: "Fix decimal scaling in reward calculations"
   - Example: "Prioritize liquidator rewards over protocol fees"
   - Example: "Include earned yield in collateral valuation"

### Short-term Improvements (Medium)
1. [List medium-priority enhancements]
   - Example: "Increase minimum collateral to account for liquidation costs"
   - Example: "Add swap fees during liquidation"

### Long-term Enhancements (Low)
1. [List optimization opportunities]
   - Example: "Dynamic protocol fees based on position size"

## Checklist Results

Based on `checklist.md`:

- [x] **Decimal precision:** Rewards scaled to collateral decimals ✓/✗
- [x] **Fee priority:** Liquidator paid before protocol fees ✓/✗
- [x] **Fee economics:** Protocol fee <30% of bonus ✓/✗
- [x] **Minimum collateral:** Accounts for liquidation costs ✓/✗
- [x] **Yield inclusion:** Earned yield in collateral value ✓/✗
- [x] **Swap fees:** Charged during liquidation if applicable ✓/✗
- [x] **Self-liquidation:** Prevented via checks or delays ✓/✗

## Economic Analysis

### Liquidation Profitability Model

**Assumptions:**
- Gas price: [X gwei]
- Liquidation gas cost: [Y gas]
- Current bonus: [Z%]
- Protocol fee: [A%]

**Net Liquidator Reward:**
```
Bonus = debt × Z%
Protocol fee = bonus × A%
Net reward = bonus - protocol_fee
Gas cost = X gwei × Y gas = B USD
Net profit = net_reward - gas_cost
```

**Minimum Profitable Position:**
```
Gas cost = B USD
Net reward rate = Z% - (Z% × A%) = C%
Minimum debt = B / C%
```

**Current minimum:** [D USD]
**Recommended minimum:** [B / C% USD]

### Decimal Error Impact

**USDC (6 decimals) example:**
```
Incorrect: reward = (debt_18_decimals × 110) / 1e18 = ~0 USDC
Correct: reward = (debt_6_decimals × 110) / 100 = actual USDC
```

**Impact:** [All USDC liquidations fail / X% of positions affected]

### Fee Priority Impact

**Protocol fee first:**
```
Collateral: 1100
Protocol fee (50%): 550
Remaining: 550
Liquidator reward: 550 - 1000 = negative (fails)
```

**Liquidator paid first:**
```
Bonus: 1100
Liquidator reward: 100
Protocol fee (50% of reward): 50
Net liquidator: 50 (positive)
```

### Yield Inclusion Impact

**Without yield:**
```
User deposits: 1000
Earned yield: 200
Collateral value (incorrect): 1000
Debt: 900
Health: 1000/900 = 111% (liquidatable at 120%)
Result: unfair liquidation
```

**With yield:**
```
User deposits: 1000
Earned yield: 200
Collateral value (correct): 1200
Debt: 900
Health: 1200/900 = 133% (healthy)
Result: fair
```

## Testing Recommendations

### Unit Tests
- [ ] Liquidation reward calculation with 6/8/18 decimal tokens
- [ ] Fee priority (liquidator paid first)
- [ ] Protocol fee edge cases (0%, 50%, 100%)
- [ ] Minimum collateral threshold liquidation profitability
- [ ] Yield inclusion in collateral valuation
- [ ] Self-liquidation prevention
- [ ] Oracle update delays

### Integration Tests
- [ ] Multi-token liquidations with different decimals
- [ ] Fee distribution to liquidator vs protocol
- [ ] Yield vault integration in liquidations
- [ ] Oracle manipulation attempts
- [ ] Liquidation profitability at various gas prices

### Economic Simulations
- [ ] Liquidation profitability across position sizes
- [ ] Protocol fee impact on liquidation volume
- [ ] Yield accrual impact on liquidation rates
- [ ] Minimum collateral sufficiency under volatility

## Appendix

### Decimal Conversion Reference

| Token | Decimals | Example Amount | Raw Value |
|-------|----------|----------------|-----------|
| USDC  | 6        | 1000 USDC      | 1000e6    |
| DAI   | 18       | 1000 DAI       | 1000e18   |
| WBTC  | 8        | 1 WBTC         | 1e8       |

**Scaling formula:**
```
amount_in_token_B = amount_in_token_A × (10^decimals_B) / (10^decimals_A)
```

### Gas Cost Analysis

**Ethereum mainnet (50 gwei, $2000 ETH):**
- Simple liquidation: 150k gas = $15
- Complex (with swaps): 300k gas = $30

**L2 (Arbitrum/Optimism):**
- Simple: 150k gas = $0.50
- Complex: 300k gas = $1

**Adjust minimum position sizes accordingly**

### Fee Structure Best Practices

**Liquidation bonus:** 5-15% (incentive for liquidators)
**Protocol fee:** <10% of bonus (maintain profitability)
**Net liquidator reward:** >gas cost + buffer

**Example:**
- Bonus: 10%
- Protocol fee: 5% of bonus = 0.5% of debt
- Net liquidator: 9.5% of debt
- Minimum position: gas_cost / 9.5%

### Self-Liquidation Prevention

**Method 1:** Direct check
```solidity
require(msg.sender != user, "Cannot self-liquidate");
```

**Method 2:** Oracle delay
```solidity
require(
    block.timestamp > lastOracleUpdate[msg.sender] + ORACLE_DELAY,
    "Oracle delay"
);
```

**Method 3:** Whitelist liquidators
```solidity
require(isAuthorizedLiquidator[msg.sender], "Not authorized");
```
