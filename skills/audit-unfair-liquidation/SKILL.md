---
name: audit-unfair-liquidation
description: Audits Solidity liquidation mechanisms for unfair liquidation vulnerabilities including missing L2 sequencer grace periods, interest accumulation while paused, repayment paused while liquidation active, late interest/fee updates, lost positive PNL/yield during liquidation, unhealthier post-liquidation state from cherry-picking, corrupted collateral priority, borrower replacement misattribution, no LTV gap allowing immediate liquidation, interest during auctions, and no liquidation slippage protection (project)
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Unfair Liquidation Auditor

## When to Use
- Auditing liquidation fairness, L2 deployments, pause mechanisms
- User mentions: L2 sequencer, grace period, pause, interest accumulation, LTV gap, auction, slippage protection, collateral priority
- Analyzing liquidation timing, state synchronization, user protections
- Reviewing pause/unpause logic, sequencer checks, health score calculations

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-unfair-liquidation** skill to analyze this contract for unfair liquidation vulnerabilities..."

1. **Scan for liquidation fairness issues**
   - Search: `sequencer`, `paused`, `accrue`, `LTV`, `auction`, `healthFactor`, `slippage`, `collateralPriority`, `borrower`
   - Focus: sequencer checks, pause states, interest accrual, health calculations, auction mechanics

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit unfair liquidation mechanisms?
   - Can users be liquidated during sequencer downtime without grace period?
   - Does interest accumulate while repayments paused?
   - Can liquidators cherry-pick stable collateral?
   - Is there an LTV gap between borrow and liquidation?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless direct borrower harm

4. **Generate report**
   - Use deliverable template below
   - Include timing analysis and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Missing L2 sequencer grace period → users liquidated immediately when sequencer restarts
2. Interest accumulates while paused → users liquidated for interest accrued during downtime
3. Repayment paused, liquidation active → users prevented from avoiding liquidation
4. Late interest/fee updates → isLiquidatable uses stale values
5. Lost positive PNL/yield → profitable positions lose gains during liquidation
6. Unhealthier post-liquidation → liquidator cherry-picks stable collateral
7. Corrupted collateral priority → liquidation order doesn't match risk
8. Borrower replacement misattribution → original borrower repays new owner's debt
9. No LTV gap → users liquidatable immediately after borrowing
10. Interest during auction → borrowers accrue interest while being auctioned
11. No liquidation slippage protection → liquidators can't specify minimum rewards

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Missing sequencer grace period on L2, repayment paused while liquidation active, immediate liquidation after borrow (no LTV gap), **MUST be exploitable by non-privileged actors**
**High:** Interest accumulation during pause, late fee updates, cherry-picking collateral leaving unhealthier state, **MUST be exploitable by non-privileged actors**
**Medium:** Lost PNL/yield during liquidation, interest accrual during auctions, no liquidator slippage protection, **admin-only liquidation timing configuration issues with cascading borrower impact**
**Low:** Suboptimal collateral priority without security impact, **admin-only parameter issues without immediate borrower impact**

**IMPORTANT:** Admin-only liquidation timing functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Admin can trigger liquidations during sequencer downtime without grace period
- Pause mechanism controlled by admin creates asymmetry (repayment paused, liquidation active)
- LTV configuration by admin allows immediate liquidation after borrowing

## False Positives - Do NOT Flag

- L1-only deployments (no sequencer concerns)
- Protocols with explicit admin-only liquidation during pause
- Systems where pause simultaneously halts interest and liquidation
- Documented grace periods after unpause events
- Single-collateral systems (no cherry-picking possible)
- **Admin-only pause functions** (onlyOwner, onlyAdmin) where pause halts both repayment and liquidation symmetrically
- Governance-controlled LTV parameters with documented gap between borrow and liquidation thresholds
- Admin functions for grace period configuration with reasonable defaults

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, timing/fairness analysis, PoC, remediation.

## Key Principles

- **Fairness** - users must have opportunity to avoid liquidation
- **Synchronization** - pause states must be consistent (repay ↔ liquidate)
- **Grace periods** - time for users to respond after downtime
- **Health improvement** - liquidation must improve borrower health
- **Predictability** - liquidators need slippage protection

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide timing analysis (sequencer downtime windows)
- Show PoCs demonstrating unfair liquidation scenarios
- Analyze pause state interactions
- Calculate LTV gaps and health score changes

**DON'T:**
- Report intentional pause-based liquidation designs
- Flag missing features with alternative mechanisms
- Use vague terms ("might be unfair")
- Ignore documented grace period configurations
