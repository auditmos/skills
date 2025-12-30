---
name: audit-state-validation
description: Audits Solidity smart contracts for state validation vulnerabilities including unchecked 2-step ownership transfers allowing address(0) bricking, functions accepting unexpected matching or empty inputs bypassing validation, unchecked return values causing silent failures, non-existent ID manipulation corrupting state, missing access control on critical functions, inconsistent array length validation, and improper pause mechanism implementation
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# State Validation Auditor

## When to Use
- Auditing Solidity contracts for input validation and state management vulnerabilities
- User mentions: ownership, transfer, validation, access control, array length, return value, initialize, onlyOwner, pause
- Analyzing administrative functions, multi-step processes, array operations, access-controlled operations
- Reviewing state transitions, ownership mechanisms, authorization checks

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-state-validation** skill to analyze this contract for state management and validation vulnerabilities..."

1. **Scan for validation-critical operations**
   - Search: `transferOwnership`, `onlyOwner`, `initialize`, function parameters (arrays, IDs, addresses), `.call`, `.transfer`, external function calls
   - Focus: ownership transfers, access control modifiers, array operations, return value checks, pause mechanisms

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - Can attacker brick ownership or bypass access control?
   - Can state be corrupted via invalid inputs?
   - Verify no compensating validation in caller functions

4. **Generate report**
   - Use deliverable template below
   - Include PoC for each finding
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Unchecked 2-step ownership transfer → ownership bricked via address(0)
2. Unexpected matching inputs → bypassed validation (swap(tokenA, tokenA))
3. Unexpected empty inputs → bypassed logic (arrays/zero values)
4. Unchecked return values → silent failures, state inconsistencies
5. Non-existent ID manipulation → default values corrupt state
6. Missing access control → unauthorized critical operations
7. Inconsistent array length validation → OOB errors, state corruption
8. Improper pause mechanism → users prevented from self-protection

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Ownership bricking via address(0), missing access control on fund transfers/withdrawals, state corruption enabling fund extraction by **non-privileged actors**
**High:** Unauthorized state manipulation by external actors, ID corruption, array length mismatches causing DoS or incorrect state updates, **MUST be exploitable by non-privileged actors**
**Medium:** Input validation bypasses in edge cases, unchecked returns without immediate impact, pause mechanism issues, **admin-only validation issues with cascading user impact**
**Low:** View function validation issues, documented design choices, admin-only parameter validation without user impact, no security impact

**IMPORTANT:** Admin-only setter functions (onlyOwner, onlyAdmin, onlyGovernance) with missing validation are **MEDIUM or LOW severity** unless:
- Invalid parameters directly brick critical user functions
- Missing validation enables admin to rug pull or extract funds (governance attack)
- Error in admin function cascades to affect all users immediately

## False Positives - Do NOT Flag

- OpenZeppelin Ownable2Step (already secure 2-step implementation)
- Identical input validation in wrapper/library functions with explicit comments
- Unchecked returns in view/pure functions (no state change)
- **Admin-only setter functions** (onlyAdmin, onlyOwner) with missing bounds checks unless they brick user operations
- **Admin-only parameter validation** where admin is trusted and error doesn't affect users immediately
- Intentional permissionless functions (explicitly no access control)
- Array length checks in external caller functions (defense in depth acceptable)
- Admin functions with trust assumptions documented in comments

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, impact analysis, PoC showing exploitation, remediation, gas impact.

## Key Principles

- **Defense in depth** - validate all inputs, check all returns
- **Explicit over implicit** - require statements over assumptions
- **Fail-safe defaults** - revert on invalid state rather than silent corruption
- **Access control everywhere** - protect all state-changing functions appropriately

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide executable PoCs showing state corruption or unauthorized access
- Quantify impact (funds at risk, protocol DoS)
- Suggest specific modifiers (onlyOwner, onlyRole) but note generic pattern

**DON'T:**
- Report style issues without exploit path
- Flag intentional designs documented in comments
- Use vague terms ("could be vulnerable")
- Ignore context (caller validation, external protections)
