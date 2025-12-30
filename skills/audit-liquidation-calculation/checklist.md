# Liquidation Calculation Audit Checklist

Run through all items systematically before generating report. Flag violations as findings.

---

## 1. Liquidator Reward Calculation

**Pattern: Incorrect decimal precision in reward calculations**

- [ ] Liquidator reward scaled to collateral token decimals
- [ ] No hardcoded 1e18 when collateral != 18 decimals
- [ ] Reward calculations tested with 6, 8, 18 decimal tokens
- [ ] Bonus percentage applied correctly (e.g., 110% = debt * 11/10)
- [ ] No overflow/underflow in reward calculations
- [ ] Reward amount >0 for all valid liquidations

**Search terms:** `liquidationReward`, `liquidationBonus`, `calculateReward`

**Files to check:** Liquidation functions, reward calculation helpers

---

## 2. Fee Priority and Payment Order

**Pattern: Liquidator reward unprioritized, paid after other fees**

- [ ] Liquidator reward calculated before protocol fees
- [ ] Liquidator reward not reduced by subsequent fee deductions
- [ ] Protocol fees taken from remaining collateral after reward
- [ ] Fee payment order explicitly documented in code
- [ ] No edge cases where reward = 0 due to other fees

**Search terms:** `protocolFee`, fee calculation order in `liquidate()`

**Files to check:** Liquidation functions, fee distribution logic

---

## 3. Protocol Fee Economics

**Pattern: Excessive protocol fees make liquidation unprofitable**

- [ ] Protocol fee <30% of liquidation bonus
- [ ] Net liquidator reward >estimated gas costs
- [ ] Fee structure tested across position sizes (small/medium/large)
- [ ] Minimum profitable position size documented
- [ ] Fee changes require governance/timelock

**Search terms:** `PROTOCOL_FEE`, `protocolFeeRate`, fee constants

**Files to check:** Fee configuration, liquidation functions

**Analysis required:**
```
liquidation_bonus = 10%
protocol_fee = 5%
net_reward = bonus - protocol_fee = 5%
gas_cost = $20 (example)
minimum_profitable = gas_cost / net_reward = $400 position
```

---

## 4. Minimum Collateral Requirements

**Pattern: Minimum collateral doesn't account for liquidation costs**

- [ ] Minimum collateral > debt + liquidation_bonus + protocol_fee + gas_buffer
- [ ] Positions at minimum threshold profitably liquidatable
- [ ] Buffer accounts for gas price volatility (2-5x typical gas)
- [ ] Different minimums for different collateral types (if applicable)
- [ ] Liquidation ratio >100% (typically 120-150%)

**Search terms:** `minimumCollateral`, `LIQUIDATION_RATIO`, `MIN_COLLATERAL_RATIO`

**Files to check:** Borrow functions, collateral validation

**Example calculation:**
```
debt = 100 USDC
liquidation_ratio = 120%
minimum_collateral = 120 USDC

liquidation_bonus = 10% = 12 USDC
protocol_fee = 30% of bonus = 3.6 USDC
gas_cost = ~$5 = 5 USDC
net_reward = 12 - 3.6 = 8.4 USDC
profitable? 8.4 > 5 = YES
```

---

## 5. Yield and PNL Inclusion

**Pattern: Earned yield/positive PNL not included in collateral value**

- [ ] Collateral value includes deposited + earned_yield
- [ ] Positive PNL included in collateral calculations
- [ ] Yield-bearing tokens use current balance, not deposit amount
- [ ] PNL updated before liquidation checks
- [ ] No unfair liquidations due to missing yield
- [ ] Users can withdraw earned yield before liquidation

**Search terms:** `getCollateralValue`, `totalCollateral`, `isLiquidatable`, `getPNL`, `getYield`

**Files to check:** Collateral valuation functions, liquidation checks, vault integrations

**Red flags:**
- `userDeposits[user]` instead of `vault.balanceOf(user)` for yield vaults
- PNL calculation without including unrealized gains
- Liquidation checks before yield accrual updates

---

## 6. Swap Fee Application

**Pattern: Missing swap fees during liquidation**

- [ ] Swap fees charged when liquidation involves token swaps
- [ ] Fee rate consistent with non-liquidation swaps (typically 0.3%)
- [ ] Fees go to protocol treasury, not liquidator
- [ ] Fee not deducted from liquidator reward
- [ ] Swap path optimizes for best price + fees

**Search terms:** `_swap`, `swapFee`, liquidation swap operations

**Files to check:** Liquidation functions with token swaps, DEX integrations

**Note:** This is economic optimization, not critical security issue

---

## 7. Self-Liquidation Protection

**Pattern: Users can profitably self-liquidate via oracle manipulation**

- [ ] Self-liquidation restricted (`require(msg.sender != user)` or equivalent)
- [ ] Oracle updates have delays/cooldowns before liquidation allowed
- [ ] Multiple oracle sources prevent single-point manipulation
- [ ] Liquidation bonus can't be extracted by user-controlled accounts
- [ ] Time delay between oracle update and liquidation eligibility
- [ ] Self-liquidation attempts revert or penalized

**Search terms:** `liquidate`, oracle update functions, self-liquidation checks

**Files to check:** Liquidation functions, oracle integration, access controls

**Attack flow to check:**
```
1. User position becomes liquidatable
2. User calls updateOracle() (if permissionless)
3. User immediately liquidates self via alt account
4. User receives liquidation bonus
```

---

## Position Size Analysis

For each finding, calculate:

1. **Minimum profitable position**
   - Formula: `gas_cost / net_liquidator_reward_percentage`
   - Example: $20 gas / 5% reward = $400 minimum

2. **Break-even collateral ratio**
   - Formula: `debt * (1 + liquidation_costs / liquidation_bonus)`
   - Example: 100 debt * (1 + 20/10) = 300 collateral

3. **Maximum protocol fee before unprofitable**
   - Formula: `liquidation_bonus - (gas_cost / position_size)`
   - Example: 10% bonus - (20 / 1000) = 8% max fee

---

## Gas Cost Estimates (for profitability analysis)

**Ethereum mainnet:**
- Simple liquidation: ~150k gas (~$30 at 50 gwei, $2000 ETH)
- Complex liquidation (swaps): ~300k gas (~$60)

**L2s (Arbitrum/Optimism):**
- Simple liquidation: ~150k gas (~$0.50 at 0.1 gwei equivalent)
- Complex liquidation: ~300k gas (~$1)

**BSC/Polygon:**
- Simple liquidation: ~150k gas (~$0.10)
- Complex liquidation: ~300k gas (~$0.20)

Adjust minimum position sizes based on deployment chain.

---

## Common Issues to Flag

**CRITICAL:**
- [ ] Liquidator reward uses wrong decimals (hardcoded 1e18 for USDC)
- [ ] Self-liquidation allowed with user-triggered oracle updates
- [ ] Reward calculation can be 0 or overflow

**HIGH:**
- [ ] Liquidator reward paid after protocol fees (can be reduced to 0)
- [ ] Protocol fee >30% of bonus
- [ ] Yield/positive PNL not included in collateral value
- [ ] No oracle update delay before liquidation

**MEDIUM:**
- [ ] Minimum collateral doesn't account for liquidation costs
- [ ] Missing swap fees during liquidation
- [ ] Liquidation unprofitable for small positions (<$100)

**LOW:**
- [ ] Suboptimal fee distribution
- [ ] Missing documentation on fee priorities
- [ ] No minimum position size enforced

---

## False Positives to Avoid

**DO NOT flag:**
1. Trusted liquidator systems (admin/keeper bots)
2. Documented high protocol fees with alternative incentives
3. Yield claiming delays for gas optimization (if documented)
4. Manual oracle updates (admin-only, no manipulation risk)
5. Fixed rewards with profitability analysis provided

**DO flag:**
1. Trustless liquidations without profitability guarantees
2. Unclear fee priorities in code
3. Missing yield in valuation (even if documented - causes unfair liquidations)
4. Permissionless oracle updates without liquidation delays
5. Self-liquidation allowed (even if documented - enables extraction)
