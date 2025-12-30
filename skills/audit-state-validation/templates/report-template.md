# State Validation Audit Report

**Contract:** [Contract Name]
**Files Analyzed:** [List of .sol files]
**Vulnerabilities Found:** Critical: X | High: Y | Medium: Z | Low: W

---

## [SEVERITY] Vulnerability Title
**Pattern:** #[1-8]
**File:** `path/to/Contract.sol`
**Lines:** [line numbers]
**Function:** `functionName()`

### Description
[2-3 sentences explaining the validation issue and its security impact]

### Vulnerable Code
```solidity
// Actual code from contract with line numbers
45: function acceptOwnership() external {
46:     // Missing check: does NOT verify pendingOwner was set
47:     owner = pendingOwner; // Can brick to address(0)
48:     pendingOwner = address(0);
49: }
```

### Impact
- **Attack Vector:** [Direct/Indirect] - [Explanation of how attacker exploits]
- **Impact Severity:** [Ownership bricked / Unauthorized access / State corruption / DoS]
- **Affected Functions:** [List if multiple functions share this pattern]
- **Funds at Risk:** [Amount or percentage if quantifiable]

### Proof of Concept
```solidity
function testExploit() public {
    // Setup: Attacker doesn't need to be current owner
    address attacker = address(0xBad);

    // Exploit: Call acceptOwnership when pendingOwner == address(0)
    vm.prank(attacker);
    contract.acceptOwnership();

    // Result: Ownership transferred to address(0) - protocol bricked
    assert(contract.owner() == address(0));
    // All onlyOwner functions permanently inaccessible
}
```

### Remediation
```solidity
// Fixed code
function acceptOwnership() external {
    // Validate caller is the intended pendingOwner
    require(msg.sender == pendingOwner, "Not pending owner");
    require(pendingOwner != address(0), "No pending owner");

    address oldOwner = owner;
    owner = pendingOwner;
    pendingOwner = address(0);

    emit OwnershipTransferred(oldOwner, owner);
}

// Or use OpenZeppelin Ownable2Step (recommended)
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";
contract MyContract is Ownable2Step { ... }
```

**Gas Impact:** +[X] gas per call (negligible vs security improvement)

---

## Summary

### Critical Issues Requiring Immediate Attention
1. [Issue #X] - [Brief description] - [Impact: ownership bricked / funds locked / unauthorized access]
2. [Issue #Y] - [Brief description] - [Impact and affected functions]

### Recommendations
- Use OpenZeppelin Ownable2Step for secure 2-step ownership transfers
- Add input validation for all edge cases (matching inputs, empty arrays, zero addresses)
- Check return values from all external calls (especially ERC20 transfers)
- Implement access control modifiers (onlyOwner, onlyRole) on all administrative functions
- Validate array lengths match before processing multi-array operations
- Synchronize pause mechanisms (pause repayments → pause liquidations)
- Add grace periods after unpause events to protect users
- Verify ID/key existence before operations that assume valid state
- Follow Checks-Effects-Interactions (CEI) pattern for all state changes
