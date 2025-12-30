# Unfair Liquidation Vulnerability Examples

## Pattern #1: Missing L2 Sequencer Grace Period

### VULNERABLE
```solidity
contract VulnerableL2Lending {
    ISequencerUptimeFeed public sequencerFeed;

    function liquidate(address user) external {
        (, int256 answer, , , ) = sequencerFeed.latestRoundData();

        // ISSUE: Only checks if sequencer is up, no grace period
        require(answer == 0, "Sequencer down");

        // Users liquidated immediately when sequencer restarts
        // No time to respond to price changes during downtime
        require(isLiquidatable(user), "Not liquidatable");
        _executeLiquidation(user);
    }
}
```

### FIXED
```solidity
contract FixedL2Lending {
    ISequencerUptimeFeed public sequencerFeed;
    uint256 public constant GRACE_PERIOD = 1 hours;

    function liquidate(address user) external {
        (, int256 answer, uint256 startedAt, , ) = sequencerFeed.latestRoundData();

        require(answer == 0, "Sequencer down");

        // Grace period after sequencer restart
        uint256 timeSinceUp = block.timestamp - startedAt;
        require(timeSinceUp >= GRACE_PERIOD, "Grace period active");

        // Users have 1 hour to add collateral or repay after restart
        require(isLiquidatable(user), "Not liquidatable");
        _executeLiquidation(user);
    }
}
```

## Pattern #2: Interest Accumulates While Paused

### VULNERABLE
```solidity
contract VulnerableInterestAccrual {
    bool public repaymentsPaused;
    uint256 public lastAccrualTime;
    uint256 public accumulatedInterest;

    function accrueInterest() public {
        // ISSUE: Interest accrues even during pause
        uint256 elapsed = block.timestamp - lastAccrualTime;
        accumulatedInterest += calculateInterest(elapsed);
        lastAccrualTime = block.timestamp;
    }

    function repay(uint256 amount) external {
        require(!repaymentsPaused, "Repayments paused");
        // Users can't repay but interest keeps growing
        // They become liquidatable through no fault of their own
    }

    function liquidate(address user) external {
        accrueInterest();
        require(isLiquidatable(user), "Not liquidatable");
        // Liquidation allowed even during repayment pause
    }
}
```

### FIXED
```solidity
contract FixedInterestAccrual {
    bool public repaymentsPaused;
    uint256 public lastAccrualTime;
    uint256 public pauseStartTime;

    function accrueInterest() public {
        if (repaymentsPaused) {
            // Don't accrue during pause
            return;
        }

        uint256 elapsed = block.timestamp - lastAccrualTime;
        accumulatedInterest += calculateInterest(elapsed);
        lastAccrualTime = block.timestamp;
    }

    function pauseRepayments() external onlyOwner {
        repaymentsPaused = true;
        pauseStartTime = block.timestamp;
        // Interest stops accruing when pause begins
    }

    function unpauseRepayments() external onlyOwner {
        repaymentsPaused = false;
        // Reset accrual time to unpause time, skipping paused period
        lastAccrualTime = block.timestamp;
    }
}
```

## Pattern #3: Repayment Paused, Liquidation Active

### VULNERABLE
```solidity
contract VulnerableAsymmetricPause {
    bool public repaymentsPaused;
    bool public liquidationsPaused;

    function pauseRepayments() external onlyOwner {
        repaymentsPaused = true;
        // ISSUE: Liquidations remain active
        // Users have no way to defend their positions
    }

    function repay(uint256 amount) external {
        require(!repaymentsPaused, "Repayments paused");
        _processRepayment(amount);
    }

    function liquidate(address user) external {
        // Liquidation works even when repayments paused
        require(isLiquidatable(user), "Not liquidatable");
        _executeLiquidation(user);
    }
}
```

### FIXED
```solidity
contract FixedSymmetricPause {
    bool public operationsPaused;

    function pauseOperations() external onlyOwner {
        operationsPaused = true;
        // Both repayments AND liquidations paused together
    }

    function repay(uint256 amount) external {
        require(!operationsPaused, "Operations paused");
        _processRepayment(amount);
    }

    function liquidate(address user) external {
        require(!operationsPaused, "Operations paused");
        require(isLiquidatable(user), "Not liquidatable");
        _executeLiquidation(user);
    }
}
```

## Pattern #4: Late Interest/Fee Updates

### VULNERABLE
```solidity
contract VulnerableLateUpdate {
    mapping(address => uint256) public userDebt;
    uint256 public globalInterestIndex;

    // ISSUE: isLiquidatable uses stale debt values
    function isLiquidatable(address user) public view returns (bool) {
        // Uses cached debt without accruing pending interest
        return userDebt[user] > getCollateralValue(user) * 100 / 125;
    }

    function liquidate(address user) external {
        require(isLiquidatable(user), "Not liquidatable");

        // Interest accrued AFTER check - actual debt higher
        accrueInterest();
        updateUserDebt(user);

        // Liquidation may fail or behave unexpectedly
        _executeLiquidation(user);
    }
}
```

### FIXED
```solidity
contract FixedEarlyUpdate {
    mapping(address => uint256) public userDebt;
    uint256 public globalInterestIndex;

    function isLiquidatable(address user) public returns (bool) {
        // Accrue interest FIRST
        accrueInterest();
        updateUserDebt(user);

        return userDebt[user] > getCollateralValue(user) * 100 / 125;
    }

    function liquidate(address user) external {
        // isLiquidatable already accrued interest
        require(isLiquidatable(user), "Not liquidatable");
        _executeLiquidation(user);
    }
}
```

## Pattern #5: Lost Positive PNL/Yield

### VULNERABLE
```solidity
contract VulnerablePNLLoss {
    struct Position {
        uint256 collateral;
        uint256 debt;
        int256 unrealizedPnL;
        uint256 earnedYield;
    }

    function liquidate(address user) external {
        Position memory pos = positions[user];

        // ISSUE: Positive PnL and yield ignored during liquidation
        uint256 collateralToSeize = pos.debt * 105 / 100;

        // User loses their earned profits
        // Example: 1000 collateral, 800 debt, +200 PnL
        // Should only seize ~630 (800 - 200 + bonus)
        // Actually seizes 840 (ignores +200 PnL)

        _seizeCollateral(user, collateralToSeize);
        delete positions[user];
    }
}
```

### FIXED
```solidity
contract FixedPNLCredit {
    struct Position {
        uint256 collateral;
        uint256 debt;
        int256 unrealizedPnL;
        uint256 earnedYield;
    }

    function liquidate(address user) external {
        Position memory pos = positions[user];

        // Credit positive PnL and yield to effective collateral
        int256 effectiveValue = int256(pos.collateral) + pos.unrealizedPnL + int256(pos.earnedYield);

        // Calculate seizure from net position value
        uint256 debtWithBonus = pos.debt * 105 / 100;

        if (effectiveValue > int256(debtWithBonus)) {
            // User has equity - return excess
            uint256 excess = uint256(effectiveValue) - debtWithBonus;
            _returnToUser(user, excess);
        }

        _seizeCollateral(user, pos.collateral);
        delete positions[user];
    }
}
```

## Pattern #6: Unhealthier Post-Liquidation State

### VULNERABLE
```solidity
contract VulnerableCherryPick {
    struct Collateral {
        address token;
        uint256 amount;
    }

    mapping(address => Collateral[]) public userCollateral;

    // ISSUE: Liquidator chooses which collateral to seize
    function liquidate(address user, uint256 collateralIndex) external {
        Collateral storage col = userCollateral[user][collateralIndex];

        // Liquidator picks USDC (stable), leaves BTC (volatile)
        // User's remaining position is HIGHER risk than before
        _seizeCollateral(user, col.token, col.amount);
    }
}
```

### FIXED
```solidity
contract FixedCollateralPriority {
    struct Collateral {
        address token;
        uint256 amount;
        uint256 riskWeight; // Higher = riskier
    }

    mapping(address => Collateral[]) public userCollateral;

    function liquidate(address user) external {
        // Sort by risk weight descending
        // Liquidate riskiest collateral first
        Collateral[] storage cols = userCollateral[user];

        for (uint i = 0; i < cols.length; i++) {
            // Find highest risk collateral
            uint256 maxRiskIndex = _findHighestRisk(cols);

            _seizeCollateral(user, cols[maxRiskIndex].token, cols[maxRiskIndex].amount);

            // Check if enough liquidated
            if (!isLiquidatable(user)) break;
        }

        // Post-liquidation: User left with lower-risk collateral
        require(isHealthier(user), "Health must improve");
    }
}
```

## Pattern #9: No LTV Gap

### VULNERABLE
```solidity
contract VulnerableNoLTVGap {
    uint256 public constant MAX_LTV = 8000; // 80%
    uint256 public constant LIQUIDATION_THRESHOLD = 8000; // 80%

    function borrow(uint256 amount) external {
        uint256 collateralValue = getCollateralValue(msg.sender);
        uint256 newDebt = userDebt[msg.sender] + amount;

        // ISSUE: Can borrow up to exact liquidation threshold
        require(newDebt * 10000 / collateralValue <= MAX_LTV, "Exceeds LTV");

        userDebt[msg.sender] = newDebt;
    }

    function isLiquidatable(address user) public view returns (bool) {
        // Any price movement immediately triggers liquidation
        return userDebt[user] * 10000 / getCollateralValue(user) > LIQUIDATION_THRESHOLD;
    }

    // Result: Borrow at 80% LTV, any price drop = instant liquidation
    // User has zero margin for error
}
```

### FIXED
```solidity
contract FixedLTVGap {
    uint256 public constant MAX_LTV = 7500; // 75% max borrow
    uint256 public constant LIQUIDATION_THRESHOLD = 8500; // 85% liquidation

    function borrow(uint256 amount) external {
        uint256 collateralValue = getCollateralValue(msg.sender);
        uint256 newDebt = userDebt[msg.sender] + amount;

        require(newDebt * 10000 / collateralValue <= MAX_LTV, "Exceeds LTV");

        userDebt[msg.sender] = newDebt;
    }

    function isLiquidatable(address user) public view returns (bool) {
        return userDebt[user] * 10000 / getCollateralValue(user) > LIQUIDATION_THRESHOLD;
    }

    // 10% gap: User can borrow at 75%, has buffer before 85% liquidation
    // ~13% price drop needed before liquidation (not immediate)
}
```

## Pattern #10: Interest During Auction

### VULNERABLE
```solidity
contract VulnerableAuctionInterest {
    struct Auction {
        address borrower;
        uint256 startTime;
        uint256 startDebt;
    }

    mapping(uint256 => Auction) public auctions;

    function startAuction(address user) external {
        require(isLiquidatable(user), "Not liquidatable");

        auctions[nextAuctionId++] = Auction({
            borrower: user,
            startTime: block.timestamp,
            startDebt: userDebt[user] // Records debt at start
        });
    }

    function settleAuction(uint256 auctionId) external {
        Auction memory auction = auctions[auctionId];

        // ISSUE: Interest continued accruing during 24h auction
        accrueInterest();
        uint256 currentDebt = userDebt[auction.borrower];

        // currentDebt > startDebt due to interest
        // Auction proceeds may not cover inflated debt
    }
}
```

### FIXED
```solidity
contract FixedAuctionInterest {
    struct Auction {
        address borrower;
        uint256 startTime;
        uint256 frozenDebt; // Interest frozen at auction start
    }

    mapping(uint256 => Auction) public auctions;
    mapping(address => bool) public inAuction;

    function startAuction(address user) external {
        require(isLiquidatable(user), "Not liquidatable");

        // Freeze interest at auction start
        accrueInterest();
        inAuction[user] = true;

        auctions[nextAuctionId++] = Auction({
            borrower: user,
            startTime: block.timestamp,
            frozenDebt: userDebt[user]
        });
    }

    function accrueInterest() public {
        for (address user : activeUsers) {
            // Skip users in auction - their debt is frozen
            if (inAuction[user]) continue;
            _accrueForUser(user);
        }
    }

    function settleAuction(uint256 auctionId) external {
        Auction memory auction = auctions[auctionId];
        // Use frozen debt amount, not current inflated amount
        _settleLiquidation(auction.borrower, auction.frozenDebt);
        inAuction[auction.borrower] = false;
    }
}
```

## Pattern #11: No Liquidation Slippage Protection

### VULNERABLE
```solidity
contract VulnerableNoSlippage {
    function liquidate(address user, uint256 debtToCover) external {
        require(isLiquidatable(user), "Not liquidatable");

        // ISSUE: No slippage protection
        // MEV bot can sandwich this transaction
        uint256 reward = calculateReward(debtToCover);

        // Between submission and execution:
        // - Price changes
        // - Other liquidators front-run
        // - Reward changes unexpectedly

        token.transferFrom(msg.sender, address(this), debtToCover);
        collateral.transfer(msg.sender, reward);
    }
}
```

### FIXED
```solidity
contract FixedWithSlippage {
    function liquidate(
        address user,
        uint256 debtToCover,
        uint256 minReward,
        uint256 maxDebtAccepted
    ) external {
        require(isLiquidatable(user), "Not liquidatable");

        uint256 actualDebt = getActualDebt(user);
        require(actualDebt <= maxDebtAccepted, "Debt changed");

        uint256 reward = calculateReward(debtToCover);
        require(reward >= minReward, "Reward below minimum");

        // Liquidator protected from:
        // - Unexpected debt increases
        // - Reward reduction from front-running
        // - Price manipulation

        token.transferFrom(msg.sender, address(this), debtToCover);
        collateral.transfer(msg.sender, reward);
    }
}
```

## Summary: Key Protections

1. **L2 grace period:** 1 hour minimum after sequencer restart
2. **Synchronized pause:** Repayment pause = liquidation pause
3. **Interest freeze:** Stop accrual during pause and auction
4. **Early updates:** Accrue all fees before liquidation check
5. **PNL credit:** Include unrealized gains in liquidation math
6. **Health improvement:** Verify borrower health improves
7. **Risk priority:** Liquidate volatile collateral first
8. **LTV gap:** 10%+ gap between borrow and liquidation LTV
9. **Slippage protection:** Accept minReward/maxDebt parameters
