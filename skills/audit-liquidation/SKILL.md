---
name: audit-liquidation
description: Audits Solidity liquidation mechanisms for incentive structure vulnerabilities including missing liquidation rewards making trustless liquidation unprofitable, insufficient incentives for small positions causing bad debt accumulation, profitable users withdrawing collateral eliminating liquidation incentive, missing bad debt handling mechanisms, partial liquidation bypassing bad debt accounting, and lack of partial liquidation preventing whale position liquidations
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Liquidation Incentive Auditor

## When to Use
- Auditing liquidation mechanisms, collateral management, position management
- User mentions: liquidation, incentive, reward, bonus, bad debt, partial liquidation, whale positions, insurance fund, socialization
- Analyzing liquidation profitability, bad debt handling, position sizing
- Reviewing collateral withdrawal restrictions, liquidation economics

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-liquidation** skill to analyze this contract for liquidation incentive and bad debt handling vulnerabilities..."

1. **Scan for liquidation operations**
   - Search: `liquidate`, `liquidationBonus`, `liquidationReward`, `badDebt`, `insuranceFund`, `partialLiquidation`, `minPositionSize`
   - Focus: liquidation rewards, position minimums, collateral withdrawals, bad debt handling

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit liquidation incentive issues?
   - Is liquidation profitable for trustless actors?
   - Can small positions accumulate as bad debt?
   - Can users withdraw collateral while maintaining underwater positions?
   - Is bad debt handled properly?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless systemic bad debt risk

4. **Generate report**
   - Use deliverable template below
   - Include economic analysis and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. No liquidation incentive → trustless liquidation unprofitable, positions remain underwater
2. No incentive for small positions → dust positions accumulate, protocol insolvent
3. Collateral withdrawal with positive PNL → removes liquidation incentive, positions unliquidatable
4. No bad debt mechanism → insolvent positions have no recovery path
5. Partial liquidation bypasses bad debt → liquidators extract value, protocol absorbs loss
6. No partial liquidation → whale positions exceed liquidator capacity, remain underwater

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** No liquidation incentive causing systemic bad debt accumulation, collateral withdrawal eliminating liquidation possibility, **MUST be exploitable by non-privileged actors**
**High:** Missing bad debt handling mechanism, partial liquidation bypassing bad debt accounting, insufficient incentives for small positions, **MUST be exploitable by non-privileged actors**
**Medium:** Suboptimal liquidation rewards reducing liquidation speed, missing partial liquidation for large positions, **admin-only liquidation configuration issues with cascading bad debt risk**
**Low:** Inefficient reward structures without security impact, **admin-only parameter issues without immediate bad debt impact**

**IMPORTANT:** Admin-only liquidation functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Invalid reward parameters cause systemic liquidation failure and protocol insolvency
- Missing validation enables admin to manipulate liquidation incentives for personal gain
- Configuration errors directly lead to bad debt accumulation affecting all users

## False Positives - Do NOT Flag

- Protocols with trusted liquidators (not trustless)
- Minimum position sizes with explicit documentation
- Protocols with overcollateralization requirements preventing withdrawals
- Bad debt handling via documented manual admin intervention
- Fixed liquidation rewards with analysis showing profitability
- **Admin-only liquidation reward setters** (onlyOwner, onlyAdmin) in protocols with trusted admin and documented governance
- Governance-controlled incentive parameters with timelock and analysis showing profitability
- Admin functions for insurance fund management where bad debt socialization is documented

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, economic impact analysis, PoC showing unprofitable liquidation or bad debt accumulation, remediation, gas impact.

## Key Principles

- **Profitability** - liquidation must be profitable vs gas costs for trustless actors
- **Minimum viability** - enforce minimums to ensure liquidation profitability
- **Collateral lock** - prevent withdrawals that eliminate liquidation incentive
- **Bad debt handling** - insurance fund or socialization for insolvent positions
- **Scalability** - partial liquidation for positions exceeding liquidator capacity

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide economic analysis (gas costs vs rewards)
- Show PoCs demonstrating unprofitable liquidations
- Quantify bad debt accumulation potential
- Calculate minimum profitable position sizes

**DON'T:**
- Report intentional trusted liquidator designs
- Flag missing features with alternative mechanisms in place
- Use vague terms ("might be unprofitable")
- Ignore gas cost context and liquidation economics
