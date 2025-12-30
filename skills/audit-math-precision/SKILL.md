---
name: audit-math-precision
description: Audits Solidity smart contracts for arithmetic precision vulnerabilities including division-before-multiplication causing value loss, small amounts rounding to zero enabling fee bypass, token decimal mismatches in multi-asset pools, unsafe downcasts truncating storage values, incorrect rounding direction leaking protocol fees, inverted oracle price pairs, hardcoded decimal assumptions, and time unit confusion in interest calculations
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Math Precision Auditor

## When to Use
- Auditing Solidity contracts for arithmetic vulnerabilities
- User mentions: precision, decimals, rounding, overflow, downcasting, division, multiplication, fees, rewards
- Analyzing token protocols, DeFi systems, AMMs, lending protocols
- Reviewing calculations involving multiple tokens with different decimals

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-math-precision** skill to analyze this contract for arithmetic and precision vulnerabilities..."

1. **Scan for arithmetic operations**
   - Search: `* / % **` operators, type casting, `unchecked` blocks
   - Focus: fee calculations, reward distributions, token conversions, oracle price usage

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate findings**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Verify exploitability by non-privileged actors (not just style issues)
   - Calculate impact (% loss or USD value)
   - Check for compensating protections
   - Downgrade severity if admin-only unless severe impact

4. **Generate report**
   - Use deliverable template below
   - Include PoC for each finding
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Division before multiplication → precision loss
2. Small amounts round to zero → fee bypass
3. Token decimal mismatches → magnitude errors
4. Unsafe downcasting → truncation
5. Wrong rounding direction → protocol value leak
6. Inverted oracle pairs → incorrect calculations
7. Hardcoded decimal assumptions → breaks with different tokens
8. Time unit confusion → interest calculation errors
9. Unchecked arithmetic → silent overflows
10. Exponentiation precision loss → compound calculation errors

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Direct fund extraction, >10% value loss, no preconditions required
**High:** 1-10% value leakage, specific but achievable conditions, affects protocol solvency, **MUST be exploitable by non-privileged actors**
**Medium:** <1% precision loss in edge cases, requires privileged actors or specific conditions
**Low:** Gas inefficiency, view function issues, admin-only precision issues, no security impact

**IMPORTANT:** Admin-only functions (onlyOwner, onlyAdmin, onlyGovernance) with precision issues are **LOW severity** unless:
- Precision loss is severe (>10% of intended value)
- Function is called frequently in normal operations
- Error cascades to affect user funds directly

## False Positives - Do NOT Flag

- Division-before-multiplication in view functions (display only)
- Zero-rounding with explicit `require(fee > 0, ...)`
- Different decimals in isolated modules (no cross-interaction)
- Downcasts with explicit bounds check: `require(value <= type(uint96).max)`
- Documented "favor user" rounding policies
- **Admin-only functions** (onlyOwner, onlyAdmin, onlyGovernance modifiers) with minor precision issues
- Setter functions with division-before-multiplication where values are large enough to avoid practical loss
- One-time initialization functions with precision quirks (not called in normal operations)

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, impact analysis, PoC, remediation, gas impact.

## Key Principles

- **Multiply first, divide last** - minimize truncation impact
- **Protocol-favoring rounding** - round fees up, withdrawals down
- **Explicit decimals** - never assume 18 decimals
- **Validate minimums** - prevent rounding-to-zero attacks

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide executable PoCs
- Quantify potential loss
- Group similar issues

**DON'T:**
- Report style issues
- Flag intentional designs without exploit path
- Use vague terms ("might be vulnerable")
- Ignore context
