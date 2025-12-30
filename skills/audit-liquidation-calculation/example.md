# Liquidation Calculation Vulnerability Examples

Vulnerable and secure code patterns.

---

## 1. Incorrect Liquidator Reward Calculation

### ❌ Vulnerable: Hardcoded 1e18 for USDC collateral

```solidity
contract VulnerableLiquidation {
    IERC20 public collateralToken; // USDC (6 decimals)
    IERC20 public debtToken; // DAI (18 decimals)

    uint256 constant LIQUIDATION_BONUS = 110; // 110%

    function liquidate(address user) external {
        uint256 collateral = collateralToken.balanceOf(user);
        uint256 debt = debtToken.balanceOf(user);

        // VULNERABLE: assumes 18 decimals for USDC
        uint256 reward = (debt * LIQUIDATION_BONUS) / 1e18;
        // If debt = 1000 DAI (1000e18), reward = 1100e0 = 0.0011 USDC!

        collateralToken.transfer(msg.sender, reward); // Transfers 0
    }
}
```

**Impact:** Liquidator receives 0 reward, liquidation unprofitable.

### ✅ Secure: Scaled to collateral decimals

```solidity
contract SecureLiquidation {
    IERC20 public collateralToken;
    IERC20 public debtToken;
    uint8 public collateralDecimals;
    uint8 public debtDecimals;

    uint256 constant LIQUIDATION_BONUS = 110; // 110%

    function liquidate(address user) external {
        uint256 collateral = collateralToken.balanceOf(user);
        uint256 debt = debtToken.balanceOf(user);

        // Scale debt to collateral decimals
        uint256 debtInCollateralDecimals = debt *
            (10 ** collateralDecimals) / (10 ** debtDecimals);

        uint256 reward = (debtInCollateralDecimals * LIQUIDATION_BONUS) / 100;

        require(collateral >= reward, "Insufficient collateral");
        collateralToken.transfer(msg.sender, reward);
    }
}
```

---

## 2. Unprioritized Liquidator Reward

### ❌ Vulnerable: Protocol fee paid first

```solidity
contract VulnerableFeeOrder {
    uint256 constant PROTOCOL_FEE_RATE = 5000; // 50%
    uint256 constant LIQUIDATION_BONUS = 110; // 10% bonus

    function liquidate(address user) external {
        uint256 collateral = getCollateral(user);
        uint256 debt = getDebt(user);

        // VULNERABLE: protocol fee first
        uint256 protocolFee = (collateral * PROTOCOL_FEE_RATE) / 10000;
        collateral -= protocolFee;

        uint256 liquidatorReward = collateral - debt;
        // If protocolFee large enough, liquidatorReward = 0!

        protocolFeesAccrued += protocolFee;
        transfer(msg.sender, liquidatorReward); // May be 0
        transfer(treasury, debt);
    }
}
```

**Example:**
- Collateral: 1100 USDC
- Debt: 1000 USDC
- Protocol fee: 550 USDC (50%)
- Remaining: 550 USDC
- Liquidator reward: 550 - 1000 = -450 (reverts or 0)

### ✅ Secure: Liquidator paid first

```solidity
contract SecureFeeOrder {
    uint256 constant PROTOCOL_FEE_RATE = 500; // 5%
    uint256 constant LIQUIDATION_BONUS = 110; // 10% bonus

    function liquidate(address user) external {
        uint256 collateral = getCollateral(user);
        uint256 debt = getDebt(user);

        // Calculate liquidator reward first
        uint256 bonusAmount = (debt * LIQUIDATION_BONUS) / 100;
        uint256 liquidatorReward = bonusAmount - debt;

        // Protocol fee from remaining collateral
        uint256 protocolFee = (liquidatorReward * PROTOCOL_FEE_RATE) / 10000;
        uint256 netLiquidatorReward = liquidatorReward - protocolFee;

        require(collateral >= bonusAmount, "Insufficient collateral");

        transfer(msg.sender, netLiquidatorReward);
        transfer(treasury, debt + protocolFee);
    }
}
```

---

## 3. Excessive Protocol Fee

### ❌ Vulnerable: 50% protocol fee

```solidity
contract VulnerableProtocolFee {
    uint256 constant PROTOCOL_FEE_RATE = 5000; // 50%
    uint256 constant LIQUIDATION_BONUS = 110; // 10%

    function liquidate(address user) external {
        uint256 debt = 1000e6; // 1000 USDC
        uint256 bonus = (debt * LIQUIDATION_BONUS) / 100; // 1100 USDC
        uint256 liquidatorReward = bonus - debt; // 100 USDC

        // VULNERABLE: 50% protocol fee
        uint256 protocolFee = (liquidatorReward * PROTOCOL_FEE_RATE) / 10000;
        uint256 netReward = liquidatorReward - protocolFee; // 50 USDC

        // Gas cost: ~$20
        // Net profit: $50 - $20 = $30
        // Only profitable for positions >$500
        // Positions <$500 accumulate as bad debt

        transfer(msg.sender, netReward);
    }
}
```

**Impact:** Small positions (<$500) unprofitable to liquidate.

### ✅ Secure: <10% protocol fee

```solidity
contract SecureProtocolFee {
    uint256 constant PROTOCOL_FEE_RATE = 500; // 5%
    uint256 constant LIQUIDATION_BONUS = 110; // 10%

    function liquidate(address user) external {
        uint256 debt = 1000e6; // 1000 USDC
        uint256 bonus = (debt * LIQUIDATION_BONUS) / 100; // 1100 USDC
        uint256 liquidatorReward = bonus - debt; // 100 USDC

        uint256 protocolFee = (liquidatorReward * PROTOCOL_FEE_RATE) / 10000;
        uint256 netReward = liquidatorReward - protocolFee; // 95 USDC

        // Gas cost: ~$20
        // Net profit: $95 - $20 = $75
        // Profitable for positions >$200

        transfer(msg.sender, netReward);
    }
}
```

---

## 4. Missing Liquidation Fees In Minimum Collateral

### ❌ Vulnerable: Minimum = debt

```solidity
contract VulnerableMinimum {
    uint256 constant LIQUIDATION_RATIO = 100; // 100%

    function borrow(uint256 amount) external {
        uint256 collateral = userCollateral[msg.sender];
        uint256 debt = userDebt[msg.sender];

        // VULNERABLE: minimum = debt
        require(collateral * 100 >= debt * LIQUIDATION_RATIO, "Insufficient");
        // At minimum: collateral = debt
        // Liquidation reward = 0!

        userDebt[msg.sender] += amount;
    }
}
```

**Impact:** Positions at minimum threshold unliquidatable (no reward).

### ✅ Secure: Accounts for liquidation costs

```solidity
contract SecureMinimum {
    uint256 constant LIQUIDATION_RATIO = 120; // 120%
    uint256 constant LIQUIDATION_BONUS = 110; // 10%

    function borrow(uint256 amount) external {
        uint256 collateral = userCollateral[msg.sender];
        uint256 debt = userDebt[msg.sender];

        // Minimum: collateral = 120% of debt
        require(collateral * 100 >= debt * LIQUIDATION_RATIO, "Insufficient");
        // At minimum: collateral = 1200, debt = 1000
        // Liquidation bonus = 1100
        // Liquidator reward = 1200 - 1100 = 100 (10% profit)
        // After 5% protocol fee: 95
        // Enough to cover gas + profit

        userDebt[msg.sender] += amount;
    }
}
```

---

## 5. Unaccounted Yield/PNL

### ❌ Vulnerable: Ignores earned yield

```solidity
contract VulnerableYieldTracking {
    mapping(address => uint256) public userDeposits;
    IYieldVault public vault;

    function getCollateralValue(address user) public view returns (uint256) {
        // VULNERABLE: returns deposit, ignores earned yield
        return userDeposits[user];
    }

    function isLiquidatable(address user) public view returns (bool) {
        uint256 collateral = getCollateralValue(user);
        uint256 debt = getDebt(user);
        return collateral < debt * 120 / 100;
        // User liquidated despite having sufficient collateral + yield!
    }
}
```

**Example:**
- User deposits: 1000 USDC
- Earned yield: 200 USDC
- Total value: 1200 USDC
- Debt: 1000 USDC
- `getCollateralValue()` returns 1000 (ignores yield)
- User liquidated despite 1200 > 1000 * 1.2

### ✅ Secure: Includes earned yield

```solidity
contract SecureYieldTracking {
    mapping(address => uint256) public userShares;
    IYieldVault public vault;

    function getCollateralValue(address user) public view returns (uint256) {
        // Correct: returns current balance including yield
        return vault.balanceOf(user);
    }

    function isLiquidatable(address user) public view returns (bool) {
        uint256 collateral = getCollateralValue(user); // Includes yield
        uint256 debt = getDebt(user);
        return collateral < debt * 120 / 100;
    }
}
```

---

## 6. Missing Swap Fee During Liquidation

### ❌ Vulnerable: No swap fee

```solidity
contract VulnerableSwapFee {
    function liquidate(address user) external {
        uint256 collateral = getCollateral(user); // WETH
        uint256 debt = getDebt(user); // USDC

        // VULNERABLE: no swap fee
        uint256 usdcReceived = _swap(WETH, USDC, collateral);
        // Protocol loses 0.3% revenue

        _repayDebt(user, usdcReceived);
    }

    function _swap(address from, address to, uint256 amount)
        internal returns (uint256)
    {
        // Direct swap, no fee
        return dex.swap(from, to, amount);
    }
}
```

### ✅ Secure: Charges swap fee

```solidity
contract SecureSwapFee {
    uint256 constant SWAP_FEE = 30; // 0.3%

    function liquidate(address user) external {
        uint256 collateral = getCollateral(user); // WETH
        uint256 debt = getDebt(user); // USDC

        // Charge swap fee
        uint256 swapFee = (collateral * SWAP_FEE) / 10000;
        uint256 amountToSwap = collateral - swapFee;

        uint256 usdcReceived = _swap(WETH, USDC, amountToSwap);

        protocolFees[WETH] += swapFee;
        _repayDebt(user, usdcReceived);
    }
}
```

---

## 7. Oracle Sandwich Self-Liquidation

### ❌ Vulnerable: Self-liquidation allowed

```solidity
contract VulnerableSelfLiquidation {
    IOracle public oracle;

    function updateOracle() external {
        // VULNERABLE: anyone can update oracle
        oracle.update();
    }

    function liquidate(address user) external {
        require(isLiquidatable(user), "Not liquidatable");

        uint256 collateral = getCollateral(user);
        uint256 debt = getDebt(user);
        uint256 bonus = (debt * 110) / 100;

        // VULNERABLE: no check for self-liquidation
        transfer(msg.sender, collateral);
        transfer(treasury, debt);
        // User can liquidate themselves via alt account!
    }
}
```

**Attack flow:**
1. User position becomes liquidatable (collateral = 1100, debt = 1000)
2. User calls `updateOracle()` when price favorable
3. User immediately calls `liquidate(userAddress)` from alt account
4. User receives 1100 collateral, pays 1000 debt
5. User profits 100 (10% liquidation bonus extracted)

### ✅ Secure: Prevents self-liquidation

```solidity
contract SecureSelfLiquidation {
    IOracle public oracle;
    mapping(address => uint256) public lastOracleUpdate;

    uint256 constant ORACLE_DELAY = 1 hours;

    function updateOracle() external {
        oracle.update();
        lastOracleUpdate[msg.sender] = block.timestamp;
    }

    function liquidate(address user) external {
        require(msg.sender != user, "Cannot self-liquidate");
        require(isLiquidatable(user), "Not liquidatable");

        // Prevent immediate liquidation after oracle update
        require(
            block.timestamp > lastOracleUpdate[msg.sender] + ORACLE_DELAY,
            "Oracle update delay"
        );

        uint256 collateral = getCollateral(user);
        uint256 debt = getDebt(user);
        uint256 bonus = (debt * 110) / 100;

        transfer(msg.sender, collateral);
        transfer(treasury, debt);
    }
}
```

---

## Complete Secure Liquidation Example

```solidity
contract CompleteLiquidation {
    IERC20 public collateralToken;
    IERC20 public debtToken;
    IYieldVault public vault;
    IOracle public oracle;

    uint8 public immutable collateralDecimals;
    uint8 public immutable debtDecimals;

    uint256 constant LIQUIDATION_RATIO = 120; // 120%
    uint256 constant LIQUIDATION_BONUS = 110; // 10%
    uint256 constant PROTOCOL_FEE_RATE = 500; // 5%
    uint256 constant ORACLE_DELAY = 1 hours;

    mapping(address => uint256) public userShares;
    mapping(address => uint256) public userDebt;
    mapping(address => uint256) public lastOracleUpdate;

    constructor(address _collateral, address _debt, address _vault) {
        collateralToken = IERC20(_collateral);
        debtToken = IERC20(_debt);
        vault = IYieldVault(_vault);

        collateralDecimals = IERC20Metadata(_collateral).decimals();
        debtDecimals = IERC20Metadata(_debt).decimals();
    }

    function getCollateralValue(address user) public view returns (uint256) {
        // Include earned yield
        return vault.balanceOf(user);
    }

    function isLiquidatable(address user) public view returns (bool) {
        uint256 collateral = getCollateralValue(user);
        uint256 debt = userDebt[user];

        // Scale debt to collateral decimals
        uint256 debtScaled = debt * (10 ** collateralDecimals) /
            (10 ** debtDecimals);

        return collateral * 100 < debtScaled * LIQUIDATION_RATIO;
    }

    function liquidate(address user) external {
        // Prevent self-liquidation
        require(msg.sender != user, "Cannot self-liquidate");

        // Oracle delay
        require(
            block.timestamp > lastOracleUpdate[msg.sender] + ORACLE_DELAY,
            "Oracle delay"
        );

        require(isLiquidatable(user), "Not liquidatable");

        uint256 collateral = getCollateralValue(user);
        uint256 debt = userDebt[user];

        // Scale properly
        uint256 debtScaled = debt * (10 ** collateralDecimals) /
            (10 ** debtDecimals);

        // Calculate liquidator reward FIRST
        uint256 bonusAmount = (debtScaled * LIQUIDATION_BONUS) / 100;
        uint256 liquidatorReward = bonusAmount - debtScaled;

        // Protocol fee from reward
        uint256 protocolFee = (liquidatorReward * PROTOCOL_FEE_RATE) / 10000;
        uint256 netLiquidatorReward = liquidatorReward - protocolFee;

        require(collateral >= bonusAmount, "Insufficient collateral");

        // Transfer in priority order
        vault.withdraw(user, netLiquidatorReward, msg.sender);
        vault.withdraw(user, debtScaled, address(this));
        vault.withdraw(user, protocolFee, treasury);

        userDebt[user] = 0;
        userShares[user] = 0;
    }

    function borrow(uint256 amount) external {
        uint256 collateral = getCollateralValue(msg.sender);
        uint256 debt = userDebt[msg.sender] + amount;

        // Scale debt to collateral decimals
        uint256 debtScaled = debt * (10 ** collateralDecimals) /
            (10 ** debtDecimals);

        // Enforce minimum that accounts for liquidation costs
        require(
            collateral * 100 >= debtScaled * LIQUIDATION_RATIO,
            "Insufficient collateral"
        );

        userDebt[msg.sender] = debt;
        debtToken.transfer(msg.sender, amount);
    }

    function updateOracle() external {
        oracle.update();
        lastOracleUpdate[msg.sender] = block.timestamp;
    }
}
```

**Key features:**
1. ✅ Correct decimal scaling
2. ✅ Liquidator paid first
3. ✅ Low protocol fee (5%)
4. ✅ Minimum accounts for liquidation costs
5. ✅ Includes earned yield in collateral value
6. ✅ Prevents self-liquidation
7. ✅ Oracle update delay
