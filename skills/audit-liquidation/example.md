# Liquidation Incentive Vulnerability Examples

## Pattern #1: No Liquidation Incentive

### VULNERABLE
```solidity
contract VulnerableNoIncentive {
    struct Position {
        uint256 collateral;
        uint256 debt;
    }

    mapping(address => Position) public positions;

    // ISSUE: No liquidation bonus - liquidator only gets exact debt amount
    function liquidate(address user) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        // Transfer collateral worth exactly debt amount
        uint256 collateralToLiquidator = pos.debt;
        token.transferFrom(msg.sender, address(this), pos.debt);
        token.transfer(msg.sender, collateralToLiquidator);

        // No profit for liquidator - gas costs make this unprofitable
        delete positions[user];
    }
}
```

### FIXED
```solidity
contract FixedWithIncentive {
    uint256 public constant LIQUIDATION_BONUS = 1050; // 5% bonus
    uint256 public constant BASIS_POINTS = 10000;

    struct Position {
        uint256 collateral;
        uint256 debt;
    }

    mapping(address => Position) public positions;

    function liquidate(address user) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        // Liquidator receives debt amount + 5% bonus from collateral
        uint256 liquidatorPayout = (pos.debt * LIQUIDATION_BONUS) / BASIS_POINTS;

        token.transferFrom(msg.sender, address(this), pos.debt);
        token.transfer(msg.sender, liquidatorPayout);

        // 5% bonus covers gas costs and incentivizes trustless liquidation
        delete positions[user];
    }
}
```

## Pattern #2: No Incentive To Liquidate Small Positions

### VULNERABLE
```solidity
contract VulnerableSmallPositions {
    // ISSUE: No minimum position size
    function openPosition(uint256 collateralAmount, uint256 borrowAmount) external {
        // Allows positions as small as 1 wei
        positions[msg.sender] = Position({
            collateral: collateralAmount,
            debt: borrowAmount
        });
    }

    function liquidate(address user) external {
        Position memory pos = positions[user];
        // Tiny position: debt = 0.01 ETH
        // Gas cost: ~0.05 ETH
        // Liquidation bonus: 5% = 0.0005 ETH
        // Net loss: -0.0495 ETH
        // Result: Nobody liquidates, bad debt accumulates
    }
}
```

### FIXED
```solidity
contract FixedMinimumPosition {
    uint256 public constant MIN_DEBT_SIZE = 1000e18; // 1000 tokens minimum
    uint256 public constant LIQUIDATION_BONUS = 1050; // 5%

    function openPosition(uint256 collateralAmount, uint256 borrowAmount) external {
        require(borrowAmount >= MIN_DEBT_SIZE, "Position too small");

        // Ensures liquidation profitable:
        // Min debt: 1000 tokens
        // Min bonus: 50 tokens (5%)
        // Covers gas costs, incentivizes liquidation

        positions[msg.sender] = Position({
            collateral: collateralAmount,
            debt: borrowAmount
        });
    }

    function partialRepay(uint256 amount) external {
        Position storage pos = positions[msg.sender];
        uint256 remaining = pos.debt - amount;

        // Prevent leaving dust
        require(
            remaining == 0 || remaining >= MIN_DEBT_SIZE,
            "Would leave unprofitable position"
        );

        pos.debt = remaining;
    }
}
```

## Pattern #3: Profitable User Withdraws All Collateral

### VULNERABLE
```solidity
contract VulnerableCollateralWithdrawal {
    struct Position {
        uint256 collateral;
        uint256 debt;
        int256 unrealizedPnL; // Profit/loss from trading
    }

    mapping(address => Position) public positions;

    // ISSUE: Can withdraw all collateral if in profit
    function withdrawCollateral(uint256 amount) external {
        Position storage pos = positions[msg.sender];

        // Only checks current PnL, not future risk
        require(pos.unrealizedPnL > 0, "Not profitable");
        require(amount <= pos.collateral, "Insufficient collateral");

        pos.collateral -= amount;
        token.transfer(msg.sender, amount);

        // If market reverses, position has 0 collateral but debt remains
        // Impossible to liquidate - guaranteed bad debt
    }
}
```

### FIXED
```solidity
contract FixedCollateralWithdrawal {
    uint256 public constant MIN_COLLATERAL_RATIO = 1500; // 150%
    uint256 public constant BASIS_POINTS = 10000;

    struct Position {
        uint256 collateral;
        uint256 debt;
        int256 unrealizedPnL;
    }

    mapping(address => Position) public positions;

    function withdrawCollateral(uint256 amount) external {
        Position storage pos = positions[msg.sender];

        uint256 remainingCollateral = pos.collateral - amount;

        // Must maintain minimum collateral ratio even after withdrawal
        uint256 requiredCollateral = (pos.debt * MIN_COLLATERAL_RATIO) / BASIS_POINTS;
        require(
            remainingCollateral >= requiredCollateral,
            "Would violate collateral ratio"
        );

        pos.collateral = remainingCollateral;
        token.transfer(msg.sender, amount);

        // Ensures liquidation always possible with sufficient incentive
    }
}
```

## Pattern #4: No Mechanism To Handle Bad Debt

### VULNERABLE
```solidity
contract VulnerableNoBadDebtHandling {
    uint256 public totalDebt;
    uint256 public totalCollateral;

    function liquidate(address user) external {
        Position memory pos = positions[user];
        require(pos.debt > pos.collateral, "Not underwater");

        // ISSUE: Insolvent position - debt exceeds collateral
        // Liquidator repays debt, takes all collateral
        token.transferFrom(msg.sender, address(this), pos.debt);
        token.transfer(msg.sender, pos.collateral);

        // Loss: pos.debt - pos.collateral absorbed by protocol
        // No insurance fund, no socialization
        // Protocol becomes insolvent

        totalDebt -= pos.debt;
        totalCollateral -= pos.collateral;
        delete positions[user];
    }
}
```

### FIXED
```solidity
contract FixedBadDebtHandling {
    uint256 public totalDebt;
    uint256 public totalCollateral;
    uint256 public insuranceFund;

    function liquidate(address user) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        if (pos.debt > pos.collateral) {
            // Insolvent position - bad debt exists
            uint256 badDebt = pos.debt - pos.collateral;

            // Liquidator repays collateral value only
            token.transferFrom(msg.sender, address(this), pos.collateral);
            token.transfer(msg.sender, pos.collateral);

            // Bad debt covered by insurance fund
            require(insuranceFund >= badDebt, "Insufficient insurance");
            insuranceFund -= badDebt;
            totalDebt -= pos.debt;
            totalCollateral -= pos.collateral;

            emit BadDebtSocialized(user, badDebt);
        } else {
            // Normal liquidation with bonus
            uint256 payout = (pos.debt * 1050) / 10000; // 5% bonus
            token.transferFrom(msg.sender, address(this), pos.debt);
            token.transfer(msg.sender, payout);

            totalDebt -= pos.debt;
            totalCollateral -= payout;
        }

        delete positions[user];
    }

    // Insurance fund funded by protocol fees
    function addToInsuranceFund(uint256 amount) external {
        token.transferFrom(msg.sender, address(this), amount);
        insuranceFund += amount;
    }
}
```

## Pattern #5: Partial Liquidation Bypasses Bad Debt Accounting

### VULNERABLE
```solidity
contract VulnerablePartialLiquidation {
    function partialLiquidate(address user, uint256 debtToCover) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        // ISSUE: Allows cherry-picking profitable portion
        uint256 collateralToLiquidator = (debtToCover * 1050) / 10000;

        token.transferFrom(msg.sender, address(this), debtToCover);
        token.transfer(msg.sender, collateralToLiquidator);

        // Update position
        positions[user].debt -= debtToCover;
        positions[user].collateral -= collateralToLiquidator;

        // Problem: If position is underwater (total debt > total collateral)
        // Liquidator takes profitable portion, leaves bad debt with protocol
        // Example: 1000 debt, 900 collateral
        // Liquidator liquidates 800 debt, takes 840 collateral (with bonus)
        // Remaining: 200 debt, 60 collateral - pure bad debt
    }
}
```

### FIXED
```solidity
contract FixedPartialLiquidation {
    function partialLiquidate(address user, uint256 debtToCover) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        // Check if position is insolvent
        if (pos.debt > pos.collateral) {
            // Insolvent - require full liquidation
            revert("Position insolvent, must fully liquidate");
        }

        // Only allow partial liquidation for solvent positions
        uint256 collateralToLiquidator = (debtToCover * 1050) / 10000;

        require(
            pos.collateral - collateralToLiquidator >=
            (pos.debt - debtToCover) * 1200 / 10000,
            "Would leave unhealthy position"
        );

        token.transferFrom(msg.sender, address(this), debtToCover);
        token.transfer(msg.sender, collateralToLiquidator);

        positions[user].debt -= debtToCover;
        positions[user].collateral -= collateralToLiquidator;
    }

    function fullLiquidate(address user) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        if (pos.debt > pos.collateral) {
            // Handle bad debt via insurance fund
            handleBadDebt(user, pos);
        } else {
            // Normal liquidation
            normalLiquidation(user, pos);
        }
    }
}
```

## Pattern #6: No Partial Liquidation Prevents Whale Liquidation

### VULNERABLE
```solidity
contract VulnerableNoPartialLiquidation {
    // ISSUE: Only allows full liquidation
    function liquidate(address user) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        // Whale position: 1,000,000 ETH debt
        // Individual liquidator capacity: 10,000 ETH
        // Result: Nobody can liquidate, position remains underwater

        require(
            token.balanceOf(msg.sender) >= pos.debt,
            "Insufficient balance"
        );

        token.transferFrom(msg.sender, address(this), pos.debt);
        // ... full liquidation

        delete positions[user];
    }
}
```

### FIXED
```solidity
contract FixedWithPartialLiquidation {
    uint256 public constant MIN_LIQUIDATION_AMOUNT = 1000e18;
    uint256 public constant MAX_LIQUIDATION_PERCENT = 5000; // 50% max per tx

    function partialLiquidate(address user, uint256 debtToCover) external {
        Position memory pos = positions[user];
        require(isLiquidatable(pos), "Not liquidatable");

        // Allow partial liquidation
        require(
            debtToCover >= MIN_LIQUIDATION_AMOUNT,
            "Amount too small"
        );

        require(
            debtToCover <= (pos.debt * MAX_LIQUIDATION_PERCENT) / 10000,
            "Exceeds max liquidation percent"
        );

        // Ensure position remains healthy or fully liquidated
        uint256 remainingDebt = pos.debt - debtToCover;
        if (remainingDebt > 0) {
            require(!isInsolvent(pos), "Must fully liquidate insolvent");
        }

        uint256 collateralToLiquidator = (debtToCover * 1050) / 10000;

        token.transferFrom(msg.sender, address(this), debtToCover);
        token.transfer(msg.sender, collateralToLiquidator);

        positions[user].debt -= debtToCover;
        positions[user].collateral -= collateralToLiquidator;

        // Multiple liquidators can now participate in whale liquidation
        // Each takes up to 50%, position gradually brought back to health
    }
}
```

## Summary: Key Protections

1. **Liquidation bonus:** 5-10% bonus ensures profitability vs gas costs
2. **Minimum positions:** Enforce minimums (e.g., 1000 tokens) to ensure liquidation viability
3. **Collateral locks:** Maintain minimum collateral ratio even in profit
4. **Insurance fund:** Protocol fees fund bad debt coverage
5. **Bad debt accounting:** Force full liquidation when insolvent
6. **Partial liquidation:** Enable for whale positions with proper constraints

## Economic Analysis Example

**Without Incentive:**
- Gas cost: ~200k gas × 50 gwei = 0.01 ETH = $30
- Collateral seized: 1.0 ETH = $3000
- Debt repaid: 1.0 ETH = $3000
- Net profit: $0 - $30 = -$30 loss
- Result: Unprofitable, positions unliquidated

**With 5% Bonus:**
- Gas cost: $30
- Collateral seized: 1.05 ETH = $3150
- Debt repaid: 1.0 ETH = $3000
- Net profit: $150 - $30 = $120 profit
- Result: Profitable, incentivizes trustless liquidation
