# Reentrancy Security Audit Report

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

[Detailed explanation of the reentrancy vulnerability]

**Vulnerable Code:**

```solidity
function vulnerableFunction() external {
    // Highlight the problematic call order
    externalCall(); // External call
    stateVariable = newValue; // State update after call - ISSUE
}
```

**Attack Flow:**

1. [Attacker calls vulnerable function]
2. [Function performs checks]
3. [Function makes external call at line X]
4. [Attacker's callback triggered (fallback/receive/tokensReceived/onERC721Received)]
5. [Attacker reenters function Y]
6. [Shared state variable Z still has old value]
7. [Attacker exploits stale state to extract funds]
8. [Original function continues and updates state]
9. [Result: funds drained/double-spent]

**Proof of Concept:**

```solidity
contract ReentrancyAttack {
    VulnerableContract target;
    uint256 attackCount;

    function attack() external payable {
        target.deposit{value: 1 ether}();
        target.withdraw(1 ether);
    }

    // Callback triggered during withdraw
    receive() external payable {
        if (attackCount < 10 && address(target).balance > 0) {
            attackCount++;
            target.withdraw(1 ether); // Reenter
        }
    }
}
```

**Test Case:**

```solidity
function testReentrancyExploit() public {
    // Setup
    target.deposit{value: 10 ether}(attacker);

    // Execute attack
    vm.prank(attacker);
    attackContract.attack{value: 1 ether}();

    // Verify: attacker withdrew more than deposited
    assertGt(attacker.balance, 1 ether);
    // Expected: 1 ether, Actual: 10+ ether
}
```

**Impact Analysis:**

**Direct Impact:**
- [Fund draining - quantify amount]
- [State corruption - describe affected variables]

**Systemic Impact:**
- [Protocol insolvency if widely exploited]
- [User fund loss]

**Affected Functions:**
- [List all functions sharing vulnerable state]

**Remediation:**

```solidity
// Option 1: CEI Pattern
function fixedWithCEI() external {
    // Checks
    require(balances[msg.sender] >= amount, "Insufficient");

    // Effects (state changes FIRST)
    balances[msg.sender] -= amount;

    // Interactions (external calls LAST)
    token.transfer(msg.sender, amount);
}

// Option 2: Reentrancy Guard
function fixedWithGuard() external nonReentrant {
    require(balances[msg.sender] >= amount, "Insufficient");

    token.transfer(msg.sender, amount);
    balances[msg.sender] -= amount;
}

// Option 3: Both (recommended)
function fixedWithBoth() external nonReentrant {
    require(balances[msg.sender] >= amount, "Insufficient");

    balances[msg.sender] -= amount; // CEI
    token.transfer(msg.sender, amount);
}
```

**Recommendations:**
1. [Primary fix - e.g., "Apply nonReentrant modifier to all state-changing functions"]
2. [Secondary fix - e.g., "Reorder operations to follow CEI pattern"]
3. [Defense in depth - e.g., "Add both nonReentrant and CEI pattern"]

**Gas Impact:** [Estimated additional gas cost for fix]
- nonReentrant: ~2,100 gas per call
- State reordering: 0 gas (optimization)

---

### [SEVERITY] Finding #X: [Next Vulnerability]

[Repeat above structure for each finding]

---

## Severity Definitions

**Critical:** State updates after external calls enabling direct fund draining, cross-function reentrancy manipulating critical shared state without guards.

**High:** Token transfer reentrancy without nonReentrant protection on functions managing funds, read-only reentrancy enabling price manipulation for liquidations.

**Medium:** Partial CEI violations with limited impact, missing nonReentrant on non-critical state-changing functions, cross-function reentrancy on low-value operations.

**Low:** View function reentrancy without exploitable impact, theoretical reentrancy with no viable attack vector, reentrancy on internal functions.

## Recommendations Summary

### Immediate Actions (Critical/High)
1. [List critical fixes]
   - Example: "Apply nonReentrant to withdraw(), transfer(), and deposit()"
   - Example: "Reorder state updates before external calls in withdraw()"

### Short-term Improvements (Medium)
1. [List medium-priority enhancements]
   - Example: "Add nonReentrant to remaining state-changing functions"
   - Example: "Document view function limitations during reentrancy"

### Long-term Enhancements (Low)
1. [List optimization opportunities]
   - Example: "Audit all external integrations for callback patterns"

## Checklist Results

Based on `checklist.md`:

- [x] **CEI pattern:** State changes before external calls ✓/✗
- [x] **NonReentrant modifiers:** Applied to state-changing functions ✓/✗
- [x] **Token assumptions:** No assumptions about token behavior ✓/✗
- [x] **Cross-function analysis:** Shared state protected across functions ✓/✗
- [x] **Read-only safety:** View functions handle reentrancy safely ✓/✗

## Function Analysis

### External Call Inventory

| Function | External Call | State Updates After? | NonReentrant? | Risk |
|----------|---------------|---------------------|---------------|------|
| withdraw() | token.transfer() | Yes | No | CRITICAL |
| deposit() | token.transferFrom() | No | No | Low |
| swap() | pool.swap() | Yes | Yes | Medium |

### State Variable Access Map

| State Variable | Functions Accessing | Protected? |
|----------------|-------------------|------------|
| balances | withdraw, transfer, deposit | No |
| totalSupply | mint, burn | Yes |

### Cross-Function Reentrancy Paths

```
withdraw() → external call → attacker callback
    ↓
    reenter transfer() (same balances state)
    ↓
    exploit stale balances[msg.sender]
```

## Testing Recommendations

### Unit Tests
- [ ] Reentrancy attack on withdraw()
- [ ] Cross-function reentrancy via transfer()
- [ ] ERC777 tokensReceived callback test
- [ ] ERC721 onERC721Received callback test
- [ ] Read-only reentrancy on view functions

### Integration Tests
- [ ] Multi-step reentrancy attack
- [ ] Cross-contract reentrancy scenarios
- [ ] Callback from external protocols

### Fuzz Tests
- [ ] Random reentry points during execution
- [ ] State consistency during callbacks

## Appendix

### CEI Pattern Explanation

**Checks-Effects-Interactions Pattern:**

```solidity
function followsCEI() external {
    // 1. CHECKS - Validate inputs and conditions
    require(balances[msg.sender] >= amount, "Insufficient");
    require(amount > 0, "Zero amount");

    // 2. EFFECTS - Update state variables
    balances[msg.sender] -= amount;
    totalSupply -= amount;
    emit Withdrawal(msg.sender, amount);

    // 3. INTERACTIONS - External calls LAST
    token.transfer(msg.sender, amount);
    externalContract.notify(msg.sender, amount);
}
```

### Common Reentrancy Vectors

1. **ETH transfers:** `call`, `transfer`, `send`
2. **Token callbacks:** ERC777 `tokensReceived`, ERC721 `onERC721Received`
3. **External calls:** `call`, `delegatecall`, `staticcall`
4. **Interface calls:** Any external contract interaction

### OpenZeppelin ReentrancyGuard

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyContract is ReentrancyGuard {
    function protectedFunction() external nonReentrant {
        // Function logic
    }
}
```

### Cross-Function Reentrancy Example

```
Contract state: balance = 100

1. User calls withdraw(50)
2. withdraw() sends 50 ETH (balance still 100 in state)
3. User's receive() callback reenters transfer(50, attacker)
4. transfer() checks balance (still 100), allows transfer
5. Both operations succeed, user extracted 100 ETH
```
