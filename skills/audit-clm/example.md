# Concentrated Liquidity Manager Vulnerability Examples

## Pattern #1: Forced Unfavorable Liquidity Deployment

### VULNERABLE
```solidity
contract VulnerableCLM {
    uint256 public maxDeviation = 200; // 2%
    uint32 public twapInterval = 1800; // 30 min

    // ISSUE: rebalance() has TWAP check
    function rebalance() external {
        _checkTWAP();
        _deployLiquidity();
    }

    // ISSUE: deposit() missing TWAP check!
    function deposit(uint256 amount0, uint256 amount1) external {
        // No TWAP validation
        // MEV bot can sandwich this function

        token0.transferFrom(msg.sender, address(this), amount0);
        token1.transferFrom(msg.sender, address(this), amount1);

        // Deploys liquidity at current (manipulated) price
        _deployLiquidity();
    }

    // Attack: Sandwich deposit() to force unfavorable liquidity deployment
}
```

### FIXED
```solidity
contract FixedCLM {
    uint256 public maxDeviation = 200; // 2%
    uint32 public twapInterval = 1800; // 30 min

    function rebalance() external {
        _checkTWAP();
        _deployLiquidity();
    }

    function deposit(uint256 amount0, uint256 amount1) external {
        // TWAP check in ALL liquidity deployment functions
        _checkTWAP();

        token0.transferFrom(msg.sender, address(this), amount0);
        token1.transferFrom(msg.sender, address(this), amount1);

        _deployLiquidity();
    }

    function _checkTWAP() internal view {
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        uint160 sqrtPriceTWAP = _getSqrtTWAP();

        uint256 priceDiff = sqrtPriceX96 > sqrtPriceTWAP
            ? sqrtPriceX96 - sqrtPriceTWAP
            : sqrtPriceTWAP - sqrtPriceX96;

        require(
            priceDiff * 10000 / sqrtPriceTWAP <= maxDeviation,
            "Price deviation too high"
        );
    }
}
```

## Pattern #2: Owner Rug-Pull via TWAP Parameters

### VULNERABLE
```solidity
contract VulnerableTWAPParams {
    uint256 public maxDeviation; // No bounds
    uint32 public twapInterval; // No bounds

    // ISSUE: Owner can disable TWAP protection
    function setTWAPParams(uint256 _maxDeviation, uint32 _twapInterval) external onlyOwner {
        maxDeviation = _maxDeviation; // Can set to 10000 (100%)
        twapInterval = _twapInterval; // Can set to 1 (1 second)
    }

    // With 100% deviation or 1-second TWAP, protection is useless
    function _checkTWAP() internal view {
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        uint160 sqrtPriceTWAP = _getSqrtTWAP(); // 1-second TWAP ≈ spot price

        uint256 priceDiff = sqrtPriceX96 > sqrtPriceTWAP
            ? sqrtPriceX96 - sqrtPriceTWAP
            : sqrtPriceTWAP - sqrtPriceX96;

        // 100% deviation allows any price manipulation
        require(priceDiff * 10000 / sqrtPriceTWAP <= maxDeviation, "Too high");
    }

    // Attack:
    // 1. Owner sets maxDeviation = 10000 (100%)
    // 2. Owner coordinates with MEV bot
    // 3. MEV bot sandwiches rebalance()
    // 4. Protocol deploys at terrible price
    // 5. Owner and MEV bot split profit
}
```

### FIXED
```solidity
contract FixedTWAPParams {
    uint256 public maxDeviation;
    uint32 public twapInterval;

    // Enforce reasonable bounds
    uint256 public constant MIN_MAX_DEVIATION = 10; // 0.1%
    uint256 public constant MAX_MAX_DEVIATION = 500; // 5%
    uint32 public constant MIN_TWAP_INTERVAL = 300; // 5 minutes
    uint32 public constant MAX_TWAP_INTERVAL = 3600; // 1 hour

    function setTWAPParams(uint256 _maxDeviation, uint32 _twapInterval) external onlyOwner {
        require(
            _maxDeviation >= MIN_MAX_DEVIATION && _maxDeviation <= MAX_MAX_DEVIATION,
            "Deviation out of bounds"
        );
        require(
            _twapInterval >= MIN_TWAP_INTERVAL && _twapInterval <= MAX_TWAP_INTERVAL,
            "Interval out of bounds"
        );

        maxDeviation = _maxDeviation;
        twapInterval = _twapInterval;
    }

    // Owner cannot disable protection
    // Users can verify parameters are reasonable
}
```

## Pattern #3: Tokens Permanently Stuck

### VULNERABLE
```solidity
contract VulnerableStuckTokens {
    uint256 public tokenId; // Current Uniswap V3 position

    function rebalance() external {
        // Burn old position
        (uint256 amount0, uint256 amount1) = positionManager.decreaseLiquidity(
            INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: tokenId,
                liquidity: liquidity,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            })
        );

        // Collect tokens
        positionManager.collect(
            INonfungiblePositionManager.CollectParams({
                tokenId: tokenId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            })
        );

        // ISSUE: Rounding errors in Uniswap V3
        // Some dust tokens remain uncollected
        // These accumulate in contract over time

        // Mint new position
        (tokenId, , , ) = positionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: address(token0),
                token1: address(token1),
                fee: 3000,
                tickLower: newTickLower,
                tickUpper: newTickUpper,
                amount0Desired: token0.balanceOf(address(this)),
                amount1Desired: token1.balanceOf(address(this)),
                amount0Min: 0,
                amount1Min: 0,
                recipient: address(this),
                deadline: block.timestamp
            })
        );

        // After 1000 rebalances, stuck tokens can be significant
    }

    // No function to withdraw stuck tokens
    // Tokens permanently locked in contract
}
```

### FIXED
```solidity
contract FixedStuckTokens {
    uint256 public tokenId;

    function rebalance() external {
        // ... rebalancing logic ...
    }

    // Add sweep function to rescue stuck tokens
    function sweepTokens(address token, address to) external onlyOwner {
        // Calculate tokens that should be in contract
        uint256 expectedBalance = _getExpectedBalance(token);

        // Only allow sweeping excess (stuck from rounding)
        uint256 actualBalance = IERC20(token).balanceOf(address(this));
        require(actualBalance > expectedBalance, "No excess tokens");

        uint256 excess = actualBalance - expectedBalance;
        IERC20(token).transfer(to, excess);
    }

    function _getExpectedBalance(address token) internal view returns (uint256) {
        // Calculate tokens that should be in positions
        // Returns amount locked in Uniswap V3 position
        (,,,,,,, uint128 liquidity,,,,) = positionManager.positions(tokenId);

        // Convert liquidity to token amounts
        // Account for tokens that should be in contract
        // ...

        return expectedAmount;
    }
}
```

## Pattern #4: Stale Token Approvals

### VULNERABLE
```solidity
contract VulnerableStaleApprovals {
    INonfungiblePositionManager public positionManager;

    constructor(address _positionManager) {
        positionManager = INonfungiblePositionManager(_positionManager);

        // Initial approvals
        token0.approve(address(positionManager), type(uint256).max);
        token1.approve(address(positionManager), type(uint256).max);
    }

    // ISSUE: Updating router doesn't revoke old approvals
    function setPositionManager(address _newManager) external onlyOwner {
        positionManager = INonfungiblePositionManager(_newManager);

        // Approve new manager
        token0.approve(_newManager, type(uint256).max);
        token1.approve(_newManager, type(uint256).max);

        // Old manager still has approval!
        // If old manager compromised, can drain all tokens
    }
}
```

### FIXED
```solidity
contract FixedStaleApprovals {
    INonfungiblePositionManager public positionManager;

    constructor(address _positionManager) {
        positionManager = INonfungiblePositionManager(_positionManager);

        token0.approve(address(positionManager), type(uint256).max);
        token1.approve(address(positionManager), type(uint256).max);
    }

    function setPositionManager(address _newManager) external onlyOwner {
        address oldManager = address(positionManager);

        // Revoke old approvals FIRST
        token0.approve(oldManager, 0);
        token1.approve(oldManager, 0);

        // Update manager
        positionManager = INonfungiblePositionManager(_newManager);

        // Approve new manager
        token0.approve(_newManager, type(uint256).max);
        token1.approve(_newManager, type(uint256).max);

        emit PositionManagerUpdated(oldManager, _newManager);
    }
}
```

## Pattern #5: Retrospective Fee Application

### VULNERABLE
```solidity
contract VulnerableRetrospectiveFees {
    uint256 public protocolFeePercent = 1000; // 10%
    mapping(address => uint256) public pendingFees;

    function collectFees() external {
        // Collect fees from Uniswap V3 position
        (uint256 amount0, uint256 amount1) = positionManager.collect(...);

        // ISSUE: Protocol fee applied when collected, not when earned
        uint256 protocolAmount0 = amount0 * protocolFeePercent / 10000;
        uint256 protocolAmount1 = amount1 * protocolFeePercent / 10000;

        protocolFees0 += protocolAmount0;
        protocolFees1 += protocolAmount1;

        userFees0 += amount0 - protocolAmount0;
        userFees1 += amount1 - protocolAmount1;
    }

    // Owner can increase fee before collection
    function setProtocolFee(uint256 newFee) external onlyOwner {
        protocolFeePercent = newFee; // Can set to 50% (5000)
        // Applies to already earned but uncollected fees!
    }

    // Attack:
    // 1. Fees earned over 30 days at 10% protocol fee
    // 2. Large amount of uncollected fees
    // 3. Owner sets fee to 50%
    // 4. collectFees() takes 50% of 30 days of fees
    // 5. Users lose 40% more than expected
}
```

### FIXED
```solidity
contract FixedRetrospectiveFees {
    uint256 public protocolFeePercent = 1000; // 10%

    function collectFees() external {
        (uint256 amount0, uint256 amount1) = positionManager.collect(...);

        uint256 protocolAmount0 = amount0 * protocolFeePercent / 10000;
        uint256 protocolAmount1 = amount1 * protocolFeePercent / 10000;

        protocolFees0 += protocolAmount0;
        protocolFees1 += protocolAmount1;

        userFees0 += amount0 - protocolAmount0;
        userFees1 += amount1 - protocolAmount1;
    }

    function setProtocolFee(uint256 newFee) external onlyOwner {
        // Collect existing fees with old rate FIRST
        collectFees();

        // Then update rate
        protocolFeePercent = newFee;

        // New rate only applies to future fees
    }
}
```

## Advanced Example: Complete CLM Attack

### VULNERABLE
```solidity
contract CompleteCLMVulnerable {
    uint256 public maxDeviation = 10000; // 100% - ineffective
    uint32 public twapInterval = 1; // 1 second - ineffective

    function deposit(uint256 amount0, uint256 amount1) external {
        // Missing TWAP check
        token0.transferFrom(msg.sender, address(this), amount0);
        token1.transferFrom(msg.sender, address(this), amount1);
        _mintPosition();
    }
}

contract AttackCLM {
    CompleteCLMVulnerable clm;
    IUniswapV3Pool pool;

    function attack() external {
        // 1. Flash loan 1000 ETH
        // 2. Swap 500 ETH -> USDC to move price 10%
        pool.swap(...);

        // 3. Deposit into CLM at manipulated price
        clm.deposit(100 ether, 300000 * 1e6);
        // CLM deploys liquidity at manipulated 10% worse price

        // 4. Swap back USDC -> ETH
        pool.swap(...);

        // 5. Repay flash loan
        // 6. Profit from CLM's impermanent loss
    }
}
```

### FIXED
```solidity
contract CompleteCLMFixed {
    uint256 public maxDeviation = 200; // 2% max
    uint32 public twapInterval = 1800; // 30 minutes

    uint256 public constant MIN_MAX_DEVIATION = 10;
    uint256 public constant MAX_MAX_DEVIATION = 500;

    function deposit(uint256 amount0, uint256 amount1) external {
        _checkTWAP(); // Protection enabled

        token0.transferFrom(msg.sender, address(this), amount0);
        token1.transferFrom(msg.sender, address(this), amount1);
        _mintPosition();
    }

    function _checkTWAP() internal view {
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        uint160 sqrtPriceTWAP = _getSqrtTWAP();

        uint256 priceDiff = sqrtPriceX96 > sqrtPriceTWAP
            ? sqrtPriceX96 - sqrtPriceTWAP
            : sqrtPriceTWAP - sqrtPriceX96;

        require(
            priceDiff * 10000 / sqrtPriceTWAP <= maxDeviation,
            "Price manipulation detected"
        );
    }

    function setTWAPParams(uint256 _max, uint32 _interval) external onlyOwner {
        require(_max >= MIN_MAX_DEVIATION && _max <= MAX_MAX_DEVIATION, "Out of bounds");
        maxDeviation = _max;
        twapInterval = _interval;
    }
}
```

## Summary: Key Protections

1. **TWAP everywhere:** Check TWAP in ALL liquidity deployment functions
2. **Parameter bounds:** Enforce min/max on maxDeviation (0.1%-5%) and twapInterval (5min-1hr)
3. **Sweep function:** Allow rescuing stuck tokens from rounding errors
4. **Revoke approvals:** Zero old approvals before setting new router
5. **Collect before update:** Collect fees before changing fee structure

## TWAP Validation Template

```solidity
function _checkTWAP() internal view {
    (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();

    // Get TWAP over configured interval
    uint32[] memory secondsAgos = new uint32[](2);
    secondsAgos[0] = twapInterval;
    secondsAgos[1] = 0;

    (int56[] memory tickCumulatives, ) = pool.observe(secondsAgos);
    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
    int24 arithmeticMeanTick = int24(tickCumulativesDelta / int56(uint56(twapInterval)));

    uint160 sqrtPriceTWAP = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);

    // Check deviation
    uint256 priceDiff = sqrtPriceX96 > sqrtPriceTWAP
        ? sqrtPriceX96 - sqrtPriceTWAP
        : sqrtPriceTWAP - sqrtPriceX96;

    require(
        priceDiff * 10000 / sqrtPriceTWAP <= maxDeviation,
        "Price deviation exceeded"
    );
}
```
