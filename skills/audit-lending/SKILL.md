---
name: audit-lending
description: Audits Solidity lending and borrowing protocols for vulnerabilities including premature liquidation before default, collateral manipulation preventing liquidation, loan closure without repayment, asymmetric pause mechanisms, token disallowance blocking operations, missing grace periods, incorrect liquidation share calculations, repayments to zero address, forced loan assignments, loan state manipulation via refinancing, double debt accounting, and dust loan griefing attacks
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Lending & Borrowing Auditor

## When to Use
- Auditing lending/borrowing protocols, collateral management systems
- User mentions: lending, borrowing, liquidation, collateral, loan, repayment, refinancing, debt, grace period, pause mechanism
- Analyzing loan lifecycle: creation, repayment, liquidation, refinancing
- Reviewing collateral tracking, debt calculations, pause/unpause operations

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-lending** skill to analyze this contract for lending and borrowing protocol vulnerabilities..."

1. **Scan for lending operations**
   - Search: `liquidate`, `repay`, `borrow`, `collateral`, `loan`, `refinance`, `close`, `pause`, `unpause`
   - Focus: liquidation conditions, collateral management, debt tracking, pause mechanisms

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit lending vulnerabilities?
   - Can borrower avoid liquidation?
   - Can attacker grief lenders/borrowers?
   - Can loan state be corrupted?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless direct user fund impact

4. **Generate report**
   - Use deliverable template below
   - Include PoC for each finding
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Liquidation before default → borrowers unfairly liquidated early
2. Collateral manipulation → prevents liquidation entirely
3. Loan closure without repayment → debt written off
4. Asymmetric pause mechanism → repayments paused, liquidations active
5. Token disallow blocks existing operations → repayments/liquidations fail
6. No grace period after unpause → immediate unfair liquidations
7. Incorrect liquidation shares → collateral drained with partial repayment
8. Repayments to zero address → funds burned
9. Forced loan assignment → unwilling lenders receive loans
10. Loan state manipulation → auction cancellation extends loans indefinitely
11. Double debt subtraction → pool balance corrupted
12. Dust loan griefing → bypass minLoanSize to force small loans

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Fund loss, collateral theft, debt erasure without repayment, pool insolvency, **MUST be exploitable by non-privileged actors**
**High:** Unfair liquidation, griefing preventing operations, loan state corruption enabling extended default, **MUST be exploitable by non-privileged actors**
**Medium:** Edge case timing issues, suboptimal pause mechanisms, dust attacks with limited impact, **admin-only lending configuration issues with cascading user impact**
**Low:** Inefficient implementations without security impact, **admin-only parameter issues without immediate user impact**

**IMPORTANT:** Admin-only lending functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Invalid parameters directly enable borrowers to avoid repayment or liquidation
- Missing validation enables admin to steal user collateral or erase debt without authorization
- Pause mechanism asymmetry (admin pauses repayments but not liquidations) directly harms borrowers

## False Positives - Do NOT Flag

- Liquidation timing with documented grace periods
- Admin-only token disallow for new loans (not affecting existing)
- Intentional minimum loan sizes with explicit documentation
- Refinancing with proper accounting checks
- Pause mechanisms affecting both repayment and liquidation symmetrically
- **Admin-only collateral configuration functions** (onlyOwner, onlyAdmin) with documented trust assumptions
- Governance-controlled loan parameter updates with timelock allowing users to exit
- Admin functions for emergency pause where both repayment and liquidation are halted symmetrically

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, impact analysis, PoC showing exploitation, remediation, gas impact.

## Key Principles

- **Grace periods** - borrowers need reasonable time after defaults/unpauses
- **Symmetric pauses** - pause repayments → pause liquidations
- **Collateral integrity** - cannot be zeroed or manipulated post-creation
- **Accurate accounting** - debt tracking must be atomic and correct
- **Minimum viability** - enforce minimums to prevent griefing

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide executable PoCs showing fund loss or griefing
- Quantify impact (funds at risk, borrowers affected)
- Show timing calculations for liquidation conditions

**DON'T:**
- Report intentional design choices with documentation
- Flag gas optimizations without exploit path
- Use vague terms ("could be vulnerable")
- Ignore context (grace periods elsewhere, admin controls)
