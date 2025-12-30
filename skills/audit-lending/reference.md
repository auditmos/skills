# Lending & Borrowing Vulnerability Patterns

## Pattern #1: Liquidation Before Default
**Risk:** Borrowers liquidated before payment due dates when `paymentDefaultDuration < paymentCycleDuration`
**Detection:** Check liquidation timing logic compares against correct deadline

## Pattern #2: Collateral Manipulation Preventing Liquidation
**Risk:** Attackers overwrite collateral amounts to 0, preventing liquidation entirely
**Detection:** Verify collateral records cannot be zeroed after loan creation

## Pattern #3: Loan Closure Without Repayment
**Risk:** Calling `close()` with non-existent IDs decrements counter, marking loans as repaid without payment
**Detection:** Ensure closure requires full repayment validation

## Pattern #4: Asymmetric Pause Mechanism
**Risk:** Repayments paused while liquidations remain enabled, unfairly preventing borrowers from protecting positions
**Detection:** Verify pause affects both repayment and liquidation symmetrically

## Pattern #5: Token Disallow Blocks Existing Operations
**Risk:** Disallowing tokens prevents existing loans from being repaid/liquidated, trapping funds
**Detection:** Ensure token restrictions only apply to new loans, not existing

## Pattern #6: No Grace Period After Unpause
**Risk:** Borrowers immediately liquidated when repayments resume after pause period
**Detection:** Verify grace period exists after unpause before liquidations resume

## Pattern #7: Incorrect Liquidation Share Calculations
**Risk:** Liquidator takes collateral with insufficient repayment due to calculating shares from single position instead of total debt
**Detection:** Ensure liquidation shares calculated from total outstanding debt

## Pattern #8: Repayments Sent to Zero Address
**Risk:** Deleted loan data causes repayments to be sent to `address(0)`, burning funds
**Detection:** Verify lender address stored and validated before repayment

## Pattern #9: Forced Loan Assignment
**Risk:** Malicious actors force loans onto unwilling lenders via `buyLoan()` or similar mechanisms
**Detection:** Ensure loan transfers require lender consent or whitelist

## Pattern #10: Loan State Manipulation via Refinancing
**Risk:** Borrowers cancel auctions via refinancing to extend loans indefinitely without repayment
**Detection:** Verify refinancing has proper constraints and doesn't bypass auction resolution

## Pattern #11: Double Debt Subtraction
**Risk:** Refinancing incorrectly subtracts debt twice from pool balance, corrupting accounting
**Detection:** Ensure debt updates are atomic and idempotent

## Pattern #12: Dust Loan Griefing
**Risk:** Bypassing `minLoanSize` checks to force small loans onto lenders, making operations unprofitable
**Detection:** Verify minimum loan size enforced at all loan creation/modification points
