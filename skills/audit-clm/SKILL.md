---
name: audit-clm
description: Audits Solidity concentrated liquidity manager (CLM) protocols for vulnerabilities including forced unfavorable liquidity deployment via missing TWAP checks, owner rug-pull via TWAP parameter manipulation, tokens permanently stuck from rounding errors, stale token approvals after router updates, and retrospective fee application on previously earned rewards (project)
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Concentrated Liquidity Manager Auditor

## When to Use
- Auditing CLM protocols, Uniswap V3 position managers, liquidity management
- User mentions: CLM, concentrated liquidity, Uniswap V3, rebalance, TWAP, maxDeviation, position manager, liquidity deployment
- Analyzing liquidity rebalancing, TWAP protections, fee collection
- Reviewing router integrations, approval management

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-clm** skill to analyze this contract for concentrated liquidity manager vulnerabilities..."

1. **Scan for CLM operations**
   - Search: `rebalance`, `mint`, `addLiquidity`, `TWAP`, `maxDeviation`, `twapInterval`, `router`, `approve`, `collectFees`, `protocolFee`
   - Focus: liquidity deployment, TWAP validation, approval management, fee updates

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit CLM vulnerabilities?
   - Can liquidity be deployed without TWAP checks?
   - Can owner manipulate TWAP parameters?
   - Do rounding errors accumulate stuck tokens?
   - Are old approvals revoked on router updates?
   - Can fees be changed retrospectively?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless enables rug pull or sandwich attack

4. **Generate report**
   - Use deliverable template below
   - Include sandwich attack analysis and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Forced unfavorable liquidity deployment → missing TWAP checks in some functions allow sandwich attacks
2. Owner rug-pull via TWAP parameters → setting ineffective maxDeviation/twapInterval disables protection
3. Tokens permanently stuck → rounding errors accumulate tokens that can never be withdrawn
4. Stale token approvals → router updates don't revoke previous approvals
5. Retrospective fee application → updated fees apply to previously earned rewards

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Missing TWAP checks in liquidity deployment enabling sandwich attacks, owner can disable TWAP protection to rug users, **MUST be exploitable by non-privileged actors**
**High:** Tokens permanently stuck from rounding errors, stale approvals allowing old router to drain funds, **MUST be exploitable by non-privileged actors**
**Medium:** Retrospective fee application on earned rewards, suboptimal TWAP parameters, **admin-only rebalance configuration issues with cascading user MEV exposure**
**Low:** Gas inefficiencies in rebalancing, missing events for parameter changes, **admin-only parameter issues without immediate user impact**

**IMPORTANT:** Admin-only CLM functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Admin can disable TWAP protection to sandwich user deposits/withdrawals
- Missing validation in TWAP parameter setters enables owner rug pull
- Admin router updates leave stale approvals allowing fund drainage

## False Positives - Do NOT Flag

- Protocols with trusted admins explicitly documented
- Test environments with simplified TWAP logic
- Functions only callable by trusted contracts
- Intentional dust accumulation with withdrawal mechanism
- Manual approval management by governance
- **Admin-only rebalance functions** (onlyOwner, onlyAdmin) with TWAP checks and documented trusted operator
- Governance-controlled TWAP parameter updates with bounds validation (minDeviation, maxDeviation)
- Admin router updates that properly revoke old approvals before setting new ones

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, sandwich attack analysis, PoC, remediation.

## Key Principles

- **TWAP everywhere** - all liquidity deployment must check TWAP
- **Parameter bounds** - maxDeviation/twapInterval must have min/max limits
- **Zero dust** - no token accumulation in contracts
- **Approval hygiene** - revoke old approvals before new ones
- **Fee immutability** - fees cannot change for earned rewards

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide sandwich attack scenarios
- Show PoCs with price manipulation
- Calculate rounding error accumulation
- Identify all liquidity deployment paths

**DON'T:**
- Report missing TWAP in view functions (not exploitable)
- Flag intentional dust with withdrawal mechanism
- Ignore parameter validation (critical for protection)
- Miss indirect liquidity deployment paths
