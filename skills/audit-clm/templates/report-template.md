# Concentrated Liquidity Manager Security Audit Report

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

[Detailed explanation of the CLM vulnerability]

**Vulnerable Code:**

```solidity
function deposit() external {
    // Missing TWAP check
    _deployLiquidity();
}
```

**Sandwich Attack Analysis:**

[Analyze exploitation scenario:]
- **Attack vector:** [Price manipulation via flash loan/large swap]
- **Manipulation magnitude:** [% price moved]
- **Protocol impact:** [Impermanent loss from unfavorable deployment]
- **Attacker profit:** [Calculation of MEV extraction]

**Proof of Concept:**

```solidity
contract SandwichCLM {
    function attack() external {
        // 1. Flash loan large amount
        // 2. Swap to manipulate pool price
        // 3. Call vulnerable CLM function
        // 4. CLM deploys at manipulated price
        // 5. Swap back
        // 6. Repay flash loan
        // 7. Profit from CLM's impermanent loss
    }
}
```

**Attack Flow:**
1. [Initial state - pool price, CLM holdings]
2. [Attacker swaps X tokens to move price Y%]
3. [CLM function called, deploys liquidity at manipulated price]
4. [Attacker swaps back, price returns to normal]
5. [CLM position now has immediate impermanent loss]
6. [Attacker profit: flash loan fee vs IL extracted]

**Impact Analysis:**

**Direct Impact:**
- [Immediate IL - e.g., "5% loss on $X liquidity deployed"]
- [User fund impact - e.g., "All depositors share loss"]

**Systemic Impact:**
- [Repeated exploitation - e.g., "Every rebalance/deposit sandwichable"]
- [Economic damage - e.g., "$Y cumulative losses possible"]

**Affected Functions:**
- [List all functions deploying liquidity without TWAP]

**Remediation:**

```solidity
function deposit() external {
    // Add TWAP check before liquidity deployment
    _checkTWAP();
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
```

**Recommendations:**
1. [Primary fix - e.g., "Add TWAP check to ALL liquidity deployment functions"]
2. [Parameter bounds - e.g., "Enforce maxDeviation between 0.1%-5%"]
3. [Defense in depth - e.g., "Add time delay between deposit and deployment"]

**Gas Impact:** [Estimated additional gas cost]
- TWAP check: ~20,000 gas per call

---

### [SEVERITY] Finding #X: [Next Vulnerability]

[Repeat above structure for each finding]

---

## Severity Definitions

**Critical:** Missing TWAP checks enabling sandwich attacks on liquidity deployment, owner can disable TWAP protection via parameter manipulation.

**High:** Tokens permanently stuck from rounding errors with no rescue mechanism, stale approvals allowing compromised router to drain funds, significant loss from retrospective fee application.

**Medium:** Suboptimal TWAP parameters reducing protection effectiveness, missing events for critical parameter changes.

**Low:** Gas inefficiencies in rebalancing logic, missing view functions for transparency.

## Recommendations Summary

### Immediate Actions (Critical/High)
1. [List critical fixes]
   - Example: "Add TWAP validation to deposit() and mint() functions"
   - Example: "Enforce bounds on maxDeviation (10-500 bps) and twapInterval (300-3600s)"
   - Example: "Implement sweepTokens() function for stuck tokens"

### Short-term Improvements (Medium)
1. [List medium-priority enhancements]
   - Example: "Collect fees before fee structure updates"
   - Example: "Add events for all parameter changes"

### Long-term Enhancements (Low)
1. [List optimization opportunities]
   - Example: "Optimize gas usage in rebalancing"

## Checklist Results

Based on `checklist.md`:

- [x] **TWAP checks everywhere:** All deployment functions validate TWAP ✓/✗
- [x] **TWAP parameter bounds:** maxDeviation/twapInterval bounded ✓/✗
- [x] **No token accumulation:** No stuck tokens or sweep exists ✓/✗
- [x] **Approval revocation:** Old approvals revoked on updates ✓/✗
- [x] **Fee immutability:** Fees collected before structure changes ✓/✗

## Function Analysis

### Liquidity Deployment Functions

| Function | TWAP Check? | Risk |
|----------|-------------|------|
| rebalance() | Yes | Low |
| deposit() | No | CRITICAL |
| mint() | No | CRITICAL |
| compound() | Yes | Low |

### TWAP Parameter Configuration

- **Current maxDeviation:** [X bps]
- **Recommended range:** 10-500 bps (0.1%-5%)
- **Current twapInterval:** [X seconds]
- **Recommended range:** 300-3600 seconds (5min-1hr)

### Token Accumulation Analysis

```
Expected balance (in positions): X tokens
Actual balance (in contract): Y tokens
Stuck tokens: Y - X tokens
Estimated value: $Z
```

## Sandwich Attack Economics

### Example Scenario

**Initial State:**
- Pool: 1000 ETH / 3M USDC
- Price: 1 ETH = 3000 USDC
- CLM deploying: 100 ETH

**Attack:**
1. Attacker flash borrows 500 ETH
2. Swaps 250 ETH → USDC (price moves to 3150 USDC/ETH, +5%)
3. CLM deploys 100 ETH at manipulated 3150 price
4. Attacker swaps back USDC → ETH (price returns to 3000)
5. CLM position has 5% impermanent loss immediately

**Profit Calculation:**
- CLM impermanent loss: ~2.5 ETH (~$7,500)
- Flash loan fee: ~0.09% of 500 ETH = 0.45 ETH (~$1,350)
- Net attacker profit: ~2 ETH (~$6,000)
- Protocol/users loss: ~2.5 ETH (~$7,500)

## Testing Recommendations

### Unit Tests
- [ ] TWAP check enforcement in all deployment functions
- [ ] TWAP parameter bounds validation
- [ ] Sweep function for stuck tokens
- [ ] Approval revocation on router updates
- [ ] Fee collection before structure changes

### Integration Tests
- [ ] End-to-end sandwich attack simulation
- [ ] Parameter manipulation scenarios
- [ ] Multi-rebalance token accumulation
- [ ] Router upgrade with approval management

### Scenario Tests
- [ ] High volatility with frequent rebalances
- [ ] Owner attempts to set ineffective TWAP params
- [ ] Long-term dust accumulation over 1000 rebalances
- [ ] Router compromise with stale approvals

## Appendix

### TWAP Implementation Reference

```solidity
function _getSqrtTWAP() internal view returns (uint160 sqrtPriceX96) {
    uint32[] memory secondsAgos = new uint32[](2);
    secondsAgos[0] = twapInterval;
    secondsAgos[1] = 0;

    (int56[] memory tickCumulatives, ) = pool.observe(secondsAgos);

    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
    int24 arithmeticMeanTick = int24(tickCumulativesDelta / int56(uint56(twapInterval)));

    sqrtPriceX96 = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);
}
```

### Recommended TWAP Parameters by Pool Type

| Pool Type | maxDeviation | twapInterval | Rationale |
|-----------|-------------|--------------|-----------|
| Stablecoin/Stablecoin | 10-50 bps | 300-600s | Low volatility |
| ETH/Stablecoin | 100-200 bps | 900-1800s | Medium volatility |
| Alt/ETH | 200-500 bps | 1800-3600s | High volatility |

### Impermanent Loss Formula

For price change of x%:
```
IL = 2 * sqrt(1 + x) / (1 + x) - 1
```

Example:
- 5% price change: ~0.5% IL
- 10% price change: ~2% IL
- 25% price change: ~6% IL

### Router Upgrade Checklist

1. [ ] Collect all pending fees
2. [ ] Burn current position
3. [ ] Collect all tokens from position
4. [ ] Revoke approvals to old router
5. [ ] Set new router address
6. [ ] Approve new router
7. [ ] Mint new position
8. [ ] Emit RouterUpdated event

### Stuck Token Calculation

```solidity
function calculateStuckTokens() public view returns (uint256 token0Stuck, uint256 token1Stuck) {
    // Actual balances
    uint256 balance0 = token0.balanceOf(address(this));
    uint256 balance1 = token1.balanceOf(address(this));

    // Expected balances (in position)
    (uint256 expected0, uint256 expected1) = _getPositionAmounts(tokenId);

    // Stuck = actual - expected
    token0Stuck = balance0 > expected0 ? balance0 - expected0 : 0;
    token1Stuck = balance1 > expected1 ? balance1 - expected1 : 0;
}
```

### Fee Collection Before Update Pattern

```solidity
function setProtocolFee(uint256 newFee) external onlyOwner {
    // MUST collect existing fees with old rate first
    _collectAllFees();

    // Then update
    protocolFeePercent = newFee;

    emit ProtocolFeeUpdated(newFee);
}
```
