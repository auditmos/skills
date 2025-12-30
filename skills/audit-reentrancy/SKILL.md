---
name: audit-reentrancy
description: Audits Solidity smart contracts for reentrancy vulnerabilities including token transfer reentrancy via ERC777/callback tokens, state updates after external calls enabling draining, cross-function reentrancy manipulating shared state, and read-only reentrancy exploiting stale state during callbacks (project)
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Reentrancy Auditor

## When to Use
- Auditing external calls, token transfers, state management
- User mentions: reentrancy, ERC777, callback, CEI pattern, nonReentrant, external call, state update
- Analyzing call order, state synchronization, view functions
- Reviewing token transfer patterns, external integrations

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-reentrancy** skill to analyze this contract for reentrancy vulnerabilities..."

1. **Scan for reentrancy vectors**
   - Search: `transfer`, `call`, `delegatecall`, `external`, `nonReentrant`, `ERC777`, `tokensReceived`, `onERC721Received`
   - Focus: external calls, token transfers, state updates, callback hooks

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit the reentrancy?
   - Are state updates after external calls?
   - Can callbacks reenter different functions?
   - Do view functions read stale state during reentrancy?
   - Can attackers control callback timing?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless cross-function attack exists

4. **Generate report**
   - Use deliverable template below
   - Include attack flow and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Token transfer reentrancy → ERC777/callback tokens allow reentrancy during transfers
2. State update after external call → transfer-before-update pattern enables draining
3. Cross-function reentrancy → reenter different functions to manipulate shared state
4. Read-only reentrancy → read stale state during reentrancy for profit

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** State updates after external calls enabling direct fund draining, cross-function reentrancy manipulating critical shared state, **MUST be exploitable by non-privileged actors**
**High:** Token transfer reentrancy without nonReentrant protection, read-only reentrancy enabling price manipulation exploits, **MUST be exploitable by non-privileged actors**
**Medium:** Partial CEI violations with limited impact, missing nonReentrant on non-critical functions, admin-only CEI violations with cascading impact
**Low:** View function reentrancy without exploitable impact, theoretical reentrancy with no attack vector, admin-only CEI issues without immediate user impact

**IMPORTANT:** Admin-only functions (onlyOwner, onlyAdmin) with CEI violations are **LOW severity** unless:
- Admin function makes external calls that can reenter user-facing functions
- CEI violation enables governance attack to drain user funds
- Missing nonReentrant allows admin+user reentrancy combo attack

## False Positives - Do NOT Flag

- Internal functions (not externally callable)
- View functions reading non-critical state
- Contracts explicitly designed for trusted tokens only
- External calls with documented reentrancy safety analysis
- Functions with nonReentrant modifier properly applied

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, attack flow, PoC showing fund draining, remediation.

## Key Principles

- **CEI pattern** - Checks, Effects, Interactions (state changes before external calls)
- **Mutex protection** - nonReentrant modifier on state-changing functions
- **Token awareness** - assume any token can have callbacks
- **Cross-function analysis** - consider reentering different functions
- **Read safety** - view functions must handle reentrancy

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide complete attack flow with reentry point
- Show PoCs demonstrating fund draining
- Identify all shared state accessed
- Map cross-function reentrancy paths

**DON'T:**
- Report theoretical reentrancy without exploit path
- Flag internal functions (not exploitable)
- Ignore nonReentrant modifiers already in place
- Miss cross-function reentrancy (most common miss)
