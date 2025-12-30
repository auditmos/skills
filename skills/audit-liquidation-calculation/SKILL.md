---
name: audit-liquidation-calculation
description: Audits Solidity liquidation mechanisms for calculation vulnerabilities including incorrect liquidator reward decimals making rewards too small/large, unprioritized liquidator rewards paid after other fees removing incentive, excessive protocol fees making liquidation unprofitable, minimum collateral requirements not accounting for liquidation costs, unaccounted yield/PNL not included in collateral valuation, missing swap fees during liquidation, and oracle sandwich self-liquidation manipulation
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Liquidation Calculation Auditor

## When to Use
- Auditing liquidation reward calculations, fee distribution, collateral valuation
- User mentions: liquidation reward, protocol fee, minimum collateral, yield, PNL, self-liquidation, oracle manipulation, liquidation profitability
- Analyzing liquidation economics, fee structures, collateral calculations
- Reviewing reward priorities, decimal handling in liquidations

## Audit Workflow

**IMPORTANT: Announce skill usage at start of analysis**

Begin with: "I'm using the **audit-liquidation-calculation** skill to analyze this contract for liquidation calculation and economic vulnerabilities..."

1. **Scan for liquidation calculation operations**
   - Search: `liquidate`, `liquidationReward`, `liquidationBonus`, `protocolFee`, `collateralValue`, `minimumCollateral`, `yield`, `PNL`, `earnedYield`
   - Focus: reward calculations, fee priorities, collateral valuations, decimal handling

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit liquidation calculation issues?
   - Are liquidator rewards calculated correctly with proper decimals?
   - Are rewards paid before or after other fees?
   - Do protocol fees make liquidation unprofitable?
   - Does minimum collateral account for liquidation costs?
   - Is yield/PNL included in collateral value?
   - Are swap fees charged during liquidation?
   - Can users self-liquidate profitably via oracle manipulation?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless systemic liquidation failure

4. **Generate report**
   - Use deliverable template below
   - Include economic analysis and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Incorrect liquidator reward → decimal precision errors make rewards unusable
2. Unprioritized liquidator reward → other fees paid first, no incentive remains
3. Excessive protocol fee → 30%+ fees make liquidation unprofitable
4. Missing liquidation fees in requirements → positions unliquidatable at minimum
5. Unaccounted yield/PNL → collateral undervalued, unfair liquidations
6. No swap fee during liquidation → protocol loses revenue
7. Oracle sandwich self-liquidation → users profit from triggering oracle updates

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Liquidator rewards calculated incorrectly causing systemic liquidation failure, profitable self-liquidation via oracle manipulation, **MUST be exploitable by non-privileged actors**
**High:** Unprioritized rewards removing liquidation incentive, excessive protocol fees preventing liquidation, unaccounted yield causing unfair liquidations, **MUST be exploitable by non-privileged actors**
**Medium:** Missing swap fees during liquidation, minimum collateral not accounting for costs, **admin-only fee configuration issues with cascading liquidation impact**
**Low:** Suboptimal fee structures without security impact, **admin-only parameter issues without immediate liquidation impact**

**IMPORTANT:** Admin-only liquidation fee functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Invalid fee parameters cause systemic liquidation failure (e.g., protocol fee > liquidation bonus)
- Missing validation enables admin to drain liquidation rewards for personal gain
- Calculation errors directly lead to unprofitable liquidations and bad debt accumulation

## False Positives - Do NOT Flag

- Protocols with trusted liquidators where profitability not required
- Documented admin fee collection mechanisms
- Alternative reward structures with analysis showing profitability
- Intentional yield/PNL handling with documented rationale
- Protocols without oracle-based pricing (no oracle manipulation risk)
- **Admin-only fee setter functions** (onlyOwner, onlyAdmin) with documented validation and bounds
- Governance-controlled liquidation bonus parameters with analysis showing profitability
- Admin functions for protocol fee collection where liquidator rewards are prioritized

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, economic impact analysis, PoC showing unprofitable liquidation or exploitation, remediation, gas impact.

## Key Principles

- **Decimal precision** - liquidator rewards must use correct decimals to be spendable
- **Priority** - liquidator rewards paid first, protocol fees second
- **Profitability** - total fees < liquidation bonus to maintain incentive
- **Completeness** - minimum collateral accounts for all liquidation costs
- **Fair valuation** - yield/PNL included in collateral calculations
- **Revenue capture** - protocol charges fees on liquidation swaps
- **Manipulation resistance** - prevent profitable self-liquidation via oracle updates

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide economic analysis (fees vs rewards vs costs)
- Show PoCs demonstrating unprofitable liquidations
- Quantify reward calculation errors
- Calculate break-even points for liquidation profitability

**DON'T:**
- Report intentional design choices with documentation
- Flag missing features with alternative mechanisms in place
- Use vague terms ("might be unprofitable")
- Ignore gas costs and on-chain fee context
