# State Validation Vulnerabilities

1. **Unchecked 2-Step Ownership Transfer** - Second step doesn't verify first step was initiated, allowing attackers to brick ownership by setting to address(0)
2. **Unexpected Matching Inputs** - Functions assume different inputs but fail when receiving identical ones (e.g., swap(tokenA, tokenA))
3. **Unexpected Empty Inputs** - Empty arrays or zero values bypass critical validation logic
4. **Unchecked Return Values** - Functions don't verify return values, leading to silent failures and state inconsistencies
5. **Non-Existent ID Manipulation** - Functions accepting IDs without checking existence return default values, enabling state corruption
6. **Missing Access Control** - Critical functions like `buyLoan()` or `mintRebalancer()` lack proper authorization checks (onlyOwner, onlyRole, etc.)
7. **Inconsistent Array Length Validation** - Functions accepting multiple arrays don't validate matching lengths, causing out-of-bounds errors
8. **Improper Pause Mechanism** - Pausing repayments while allowing liquidations, or no grace period after unpause
