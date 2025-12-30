---
name: audit-auction
description: Audits Solidity auction mechanisms for manipulation vulnerabilities including self-bidding to reset auction timer, auction start during L2 sequencer downtime affecting timing fairness, insufficient auction length validation allowing very short auctions for immediate seizure, and off-by-one errors allowing seizure during active auction period (project)
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Auction Manipulation Auditor

## When to Use
- Auditing auction mechanisms, Dutch auctions, liquidation auctions
- User mentions: auction, bid, Dutch auction, auction timer, auction length, sequencer downtime, seizure
- Analyzing auction timing, bid validation, auction parameters
- Reviewing liquidation auctions, loan auctions, NFT auctions

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-auction** skill to analyze this contract for auction manipulation vulnerabilities..."

1. **Scan for auction operations**
   - Search: `auction`, `bid`, `startAuction`, `endAuction`, `auctionEnd`, `auctionLength`, `buyLoan`, `seize`, `settle`
   - Focus: timer resets, length validation, timestamp checks, sequencer integration

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit auction manipulation?
   - Can borrower reset auction by self-bidding?
   - Can auctions start during sequencer downtime?
   - Are auction lengths validated (minimum duration)?
   - Are timestamp comparisons correct (no off-by-one)?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless enables borrower manipulation

4. **Generate report**
   - Use deliverable template below
   - Include manipulation scenarios and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Self-bidding to reset auction → borrower buys own loan to restart timer indefinitely
2. Auction start during sequencer downtime → L2 sequencer issues affect fairness
3. Insufficient auction length validation → 1-second auctions allow immediate seizure
4. Auction seizure during active period → off-by-one allows premature seizure

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Self-bidding allowing infinite auction extensions, off-by-one enabling immediate seizure bypassing auction period, **MUST be exploitable by non-privileged actors**
**High:** Insufficient length validation allowing very short auctions, sequencer downtime affecting auction fairness on L2, **MUST be exploitable by non-privileged actors**
**Medium:** Suboptimal auction parameters, missing events for auction state changes, **admin-only auction configuration issues with cascading borrower impact**
**Low:** Gas inefficiencies in auction logic, missing view functions, **admin-only parameter issues without immediate borrower impact**

**IMPORTANT:** Admin-only auction functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Admin can start auctions during sequencer downtime without checks
- Missing validation in auction length setters allows admin to set 1-second auctions
- Admin auction timing configuration enables unfair seizure of borrower collateral

## False Positives - Do NOT Flag

- Protocols with explicit self-bidding allowed (documented)
- L1-only deployments (no sequencer concerns)
- Admin-only auction start (trusted operation)
- Intentional short auctions with documented rationale
- Test/mock auction contracts
- **Admin-only auction start functions** (onlyOwner, onlyAdmin) with sequencer uptime checks on L2
- Governance-controlled auction length parameters with minimum bounds validation (e.g., >= 1 hour)
- Admin functions for emergency auction settlement with documented safeguards

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, manipulation scenario, PoC, remediation.

## Key Principles

- **No self-bidding** - prevent borrower from bidding on own auction
- **Sequencer awareness** - check sequencer uptime before auction start on L2
- **Minimum duration** - enforce reasonable minimum (e.g., 1 hour)
- **Correct comparisons** - use >= not >, avoid off-by-one
- **Timer integrity** - prevent auction timer resets

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide manipulation scenarios with timing
- Show PoCs demonstrating auction abuse
- Calculate impact of timing exploits
- Map all auction state transitions

**DON'T:**
- Report intentional design choices (short auctions with rationale)
- Flag missing features with alternative mechanisms
- Ignore timestamp precision (critical for fairness)
- Miss edge cases (exact timestamp boundaries)
