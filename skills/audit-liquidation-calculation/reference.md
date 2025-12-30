# Liquidation Calculation Vulnerability Reference

Complete vulnerability patterns for liquidation calculation issues.

---

## 1. Incorrect Liquidator Reward Calculation

### Description
Decimal precision errors in liquidator reward calculations result in rewards that are too small (unusable) or too large (protocol insolvency).

### Technical Details
- Liquidator rewards must match collateral token decimals
- Common errors: hardcoding 1e18 when collateral uses 6 decimals (USDC)
- Result: reward = 0 or astronomically large value
- Makes liquidation unprofitable or impossible

### Code Pattern
```solidity
// Vulnerable: assumes 18 decimals
uint256 reward = (debt * liquidationBonus) / 1e18;

// Vulnerable: doesn't scale to collateral decimals
function calculateReward(uint256 collateral, uint256 bonus)
    returns (uint256)
{
    return collateral * bonus / 100; // Wrong precision
}
```

### Detection
- Search: `liquidationReward`, `liquidationBonus`, calculation patterns
- Check: reward calculation uses collateral token decimals
- Verify: reward precision matches token precision

### Severity
**Critical** - Prevents all liquidations or causes protocol insolvency

---

## 2. Unprioritized Liquidator Reward

### Description
Liquidator rewards paid after other fees (protocol fees, penalties) can be reduced to zero, removing liquidation incentive.

### Technical Details
- Liquidator reward must be paid first from seized collateral
- If protocol fee taken first: `reward = collateral - protocolFee - debt`
- If protocol fee large enough: `reward = 0`
- Liquidators have no incentive to liquidate

### Code Pattern
```solidity
// Vulnerable: protocol fee paid first
function liquidate(address user) external {
    uint256 collateral = getCollateral(user);
    uint256 debt = getDebt(user);

    uint256 protocolFee = collateral * protocolFeeRate / 100;
    protocolFeesAccrued += protocolFee;

    uint256 remaining = collateral - protocolFee;
    uint256 liquidatorReward = remaining - debt; // Can be 0!

    transfer(msg.sender, liquidatorReward);
}
```

### Detection
- Search: order of fee calculations in `liquidate()` functions
- Check: liquidator reward calculated before other fees
- Verify: reward amount not dependent on remaining balance after fees

### Severity
**High** - Removes liquidation incentive, accumulates bad debt

---

## 3. Excessive Protocol Fee

### Description
Protocol fees >30% of seized collateral make liquidation unprofitable after gas costs.

### Technical Details
- Liquidation bonus typically 5-15%
- Gas costs: ~100-300k gas (~$5-$50 depending on chain/prices)
- If protocol takes >30% of bonus: liquidator loses money
- Example: 10% bonus, 5% protocol fee = 5% net = unprofitable for small positions

### Code Pattern
```solidity
// Vulnerable: 50% protocol fee
uint256 constant PROTOCOL_FEE_RATE = 5000; // 50%

function liquidate(address user) external {
    uint256 collateral = getCollateral(user);
    uint256 debt = getDebt(user);
    uint256 bonus = (collateral - debt) * 110 / 100; // 10% bonus

    uint256 protocolFee = bonus * PROTOCOL_FEE_RATE / 10000; // 50% of bonus!
    uint256 liquidatorReward = bonus - protocolFee;
    // liquidatorReward too small after gas
}
```

### Detection
- Search: `PROTOCOL_FEE`, `protocolFeeRate` in liquidation functions
- Calculate: net liquidator reward after fees
- Verify: net reward > gas costs for minimum positions

### Severity
**High** - Makes liquidation unprofitable, bad debt accumulates

---

## 4. Missing Liquidation Fees In Minimum Collateral Requirements

### Description
Minimum collateral requirements don't account for liquidation costs (gas + fees), making positions unliquidatable at minimum.

### Technical Details
- Minimum collateral should be: `debt + liquidation_costs + buffer`
- Liquidation costs: gas (~$5-$50) + protocol fees
- If minimum = debt: no reward for liquidator at threshold
- Users can stay at minimum, never liquidatable profitably

### Code Pattern
```solidity
// Vulnerable: minimum = debt only
function borrow(uint256 amount) external {
    uint256 collateral = userCollateral[msg.sender];
    uint256 debt = userDebt[msg.sender];

    require(collateral >= debt, "Insufficient collateral"); // Wrong!
    // Should be: collateral >= debt * 1.2 (or higher)

    userDebt[msg.sender] += amount;
}
```

### Detection
- Search: `minimumCollateral`, collateral ratio checks in borrow functions
- Check: minimum accounts for liquidation bonus + fees + gas
- Verify: positions at minimum are profitably liquidatable

### Severity
**Medium** - Allows unliquidatable positions at minimum threshold

---

## 5. Unaccounted Yield/PNL In Collateral Valuation

### Description
Earned yield or positive PNL not included in collateral value causes unfair liquidations and lost user funds.

### Technical Details
- Collateral value should include: deposited + earned_yield + positive_PNL
- If only deposited counted: users liquidated despite being solvent
- Users lose earned yield they should be able to withdraw
- Particularly critical in yield-bearing vaults, perpetuals

### Code Pattern
```solidity
// Vulnerable: ignores earned yield
function getCollateralValue(address user) returns (uint256) {
    return userDeposits[user]; // Wrong! Missing yield
}

function isLiquidatable(address user) returns (bool) {
    uint256 collateral = getCollateralValue(user); // Understated
    uint256 debt = getDebt(user);
    return collateral < debt * LIQUIDATION_RATIO;
    // User liquidated despite having sufficient collateral + yield
}
```

### Detection
- Search: `getCollateralValue`, `isLiquidatable`, yield/PNL tracking
- Check: collateral calculation includes all value sources
- Verify: yield-bearing tokens use current balance, not deposit amount

### Severity
**High** - Causes unfair liquidations, users lose earned funds

---

## 6. No Swap Fee During Liquidation

### Description
Protocol doesn't charge swap fees when liquidation involves token swaps, losing revenue.

### Technical Details
- Liquidation often requires swapping collateral to repay debt
- Normal swaps charge 0.3% fee, liquidation swaps should too
- Lost revenue compounds over many liquidations
- Not a security issue but economic inefficiency

### Code Pattern
```solidity
// Vulnerable: no swap fee charged
function liquidate(address user) external {
    uint256 collateralAmount = getCollateral(user);
    uint256 debtAmount = getDebt(user);

    // Swap collateral token to debt token
    uint256 debtTokenReceived = _swap(
        collateralToken,
        debtToken,
        collateralAmount
    ); // No fee charged!

    _repayDebt(user, debtTokenReceived);
}
```

### Detection
- Search: swap operations in `liquidate()` functions
- Check: swap fee applied to liquidation swaps
- Verify: fee goes to protocol, not liquidator

### Severity
**Medium** - Protocol loses revenue, not security critical

---

## 7. Oracle Sandwich Self-Liquidation

### Description
Users can trigger oracle price updates to create profitable self-liquidation opportunities, extracting value from protocol.

### Technical Details
- Oracles update prices based on external triggers
- User can: 1) trigger oracle update when favorable, 2) immediately self-liquidate at new price
- Liquidation bonus paid to user's alt account
- Effectively: user extracts liquidation bonus by gaming oracle timing
- Requires oracle manipulation or just timing oracle updates

### Code Pattern
```solidity
// Vulnerable: allows self-liquidation
function liquidate(address user) external {
    require(isLiquidatable(user), "Not liquidatable");

    uint256 collateral = getCollateral(user);
    uint256 debt = getDebt(user);
    uint256 bonus = (collateral - debt) * 110 / 100;

    // No check: msg.sender != user
    // User can liquidate themselves via alt account
    transfer(msg.sender, collateral);
    transfer(treasury, debt - bonus);
}

// User flow:
// 1. Price drops, user becomes liquidatable
// 2. User calls updateOracle() to trigger price update
// 3. User calls liquidate(userAddress) from alt account
// 4. User receives liquidation bonus
```

### Detection
- Search: `liquidate()` functions, oracle update mechanisms
- Check: self-liquidation restrictions (`msg.sender != user`)
- Check: oracle update delays/cooldowns between update and liquidation
- Verify: liquidation bonus can't be gamed via oracle timing

### Severity
**Critical** - Direct value extraction via oracle manipulation

---

## Validation Checklist

For each liquidation function, verify:

1. **Decimal Precision**
   - [ ] Liquidator rewards scaled to collateral token decimals
   - [ ] No hardcoded 1e18 assumptions
   - [ ] Reward calculations tested with 6/8/18 decimal tokens

2. **Fee Priority**
   - [ ] Liquidator reward calculated first
   - [ ] Protocol fees taken from remaining balance after reward
   - [ ] Reward amount not reduced by other fees

3. **Fee Economics**
   - [ ] Protocol fee <30% of liquidation bonus
   - [ ] Net liquidator reward >gas costs for minimum positions
   - [ ] Fee structure analyzed for different position sizes

4. **Minimum Collateral**
   - [ ] Minimum accounts for: debt + liquidation_bonus + protocol_fee + gas_buffer
   - [ ] Positions at minimum threshold profitably liquidatable
   - [ ] Buffer accounts for gas price volatility

5. **Yield/PNL Inclusion**
   - [ ] Collateral value includes earned yield
   - [ ] Positive PNL included in collateral calculations
   - [ ] Yield-bearing tokens use current balance not deposit
   - [ ] PNL updated before liquidation checks

6. **Swap Fees**
   - [ ] Swap fees charged during liquidation if applicable
   - [ ] Fees go to protocol treasury
   - [ ] Fee rate consistent with non-liquidation swaps

7. **Self-Liquidation Protection**
   - [ ] Self-liquidation restricted (`msg.sender != user`)
   - [ ] Oracle update delays prevent sandwich liquidation
   - [ ] Liquidation bonus can't be extracted via timing
   - [ ] Multiple oracle price sources prevent manipulation

---

## Common False Positives

**DO NOT flag these:**

1. **Trusted liquidator systems** - Profitability not required for admin liquidators
2. **Documented fee structures** - Intentional high fees with alternative incentives
3. **Yield distribution delays** - Documented yield claiming delays for gas optimization
4. **Manual oracle updates** - Admin-only oracle updates (no user manipulation risk)
5. **Fixed reward structures** - With analysis showing profitability across position sizes

**DO flag these:**

1. **Trustless liquidators** - Without profitability guarantees
2. **Undocumented fee priorities** - Unclear whether liquidator paid first
3. **Missing yield in calculations** - Even if documented, causes unfair liquidations
4. **User-triggered oracles** - Without delays/restrictions on liquidation
5. **Self-liquidation allowed** - Even if documented, enables value extraction
