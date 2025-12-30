---
name: audit-oracle
description: Audits Solidity oracle integrations for vulnerabilities including missing stale price checks against heartbeat intervals, missing L2 sequencer uptime validation, same heartbeat for multiple feeds, assuming oracle precision, incorrect price feed addresses, unhandled oracle reverts, unhandled depeg events, oracle min/max price issues during flash crashes, using manipulable slot0 prices, price feed direction confusion, and missing circuit breaker checks (project)
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Oracle Integration Auditor

## When to Use
- Auditing oracle integrations, price feeds, Chainlink usage
- User mentions: oracle, Chainlink, price feed, TWAP, stale price, heartbeat, sequencer, depeg, circuit breaker
- Analyzing price validation, oracle failure handling, feed configuration
- Reviewing Uniswap TWAP, Chainlink feeds, custom oracles

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-oracle** skill to analyze this contract for oracle integration vulnerabilities..."

1. **Scan for oracle operations**
   - Search: `latestRoundData`, `getPrice`, `oracle`, `chainlink`, `feed`, `slot0`, `TWAP`, `observe`, `heartbeat`, `sequencer`
   - Focus: price fetching, staleness checks, error handling, feed addresses

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit oracle issues?
   - Are stale prices checked with correct heartbeats?
   - Is L2 sequencer uptime verified on L2 deployments?
   - Can oracle failures cause DoS?
   - Are depeg scenarios handled?
   - Can slot0 be manipulated?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless affects user pricing directly

4. **Generate report**
   - Use deliverable template below
   - Include price manipulation analysis and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Not checking stale prices → using outdated values during high volatility
2. Missing L2 sequencer check → prices during sequencer downtime
3. Same heartbeat for multiple feeds → wrong staleness thresholds
4. Assuming oracle precision → decimal mismatches causing errors
5. Incorrect price feed address → wrong asset pricing
6. Unhandled oracle reverts → complete DoS without fallback
7. Unhandled depeg events → using BTC/USD for compromised WBTC
8. Oracle min/max price issues → flash crashes return incorrect bounds
9. Using slot0 price → manipulable via flash loans
10. Price feed direction confusion → inverted pricing (DAI/USD vs USD/DAI)
11. Missing circuit breaker checks → not checking minAnswer/maxAnswer

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Using slot0 without TWAP enabling flash loan price manipulation, no staleness checks allowing stale price exploitation during high volatility, **MUST be exploitable by non-privileged actors**
**High:** Missing L2 sequencer checks on L2, unhandled oracle reverts causing DoS, depeg scenarios not handled, price direction confusion, **MUST be exploitable by non-privileged actors**
**Medium:** Incorrect heartbeat intervals, missing circuit breaker checks, assuming oracle decimals, incorrect feed addresses, **admin-only oracle configuration issues with cascading user impact**
**Low:** Suboptimal staleness thresholds, missing secondary oracle for redundancy, **admin-only parameter issues without immediate user impact**

**IMPORTANT:** Admin-only oracle functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Invalid oracle configuration directly prices user assets incorrectly
- Missing validation enables admin to rug pull via price manipulation
- Error cascades to affect all users immediately (e.g., stale prices in critical functions)

## False Positives - Do NOT Flag

- L1-only deployments (no sequencer concerns)
- Protocols with documented manual price updates
- Test environments with mock oracles
- View functions for display only (not used in logic)
- Oracles with explicit admin override mechanisms
- **Admin-only oracle setter functions** (onlyOwner, onlyAdmin) with minor validation issues
- Oracle feed updates by governance without immediate user pricing impact
- Parameter setters where admin is trusted and users can exit before changes take effect

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, price manipulation analysis, PoC, remediation.

## Key Principles

- **Staleness validation** - check updatedAt against heartbeat for each feed
- **Failure handling** - wrap oracle calls in try/catch
- **L2 awareness** - check sequencer uptime on L2 chains
- **Depeg monitoring** - separate feeds for wrapped assets
- **Manipulation resistance** - use TWAP, not spot prices
- **Circuit breakers** - validate prices within bounds

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide price manipulation scenarios
- Show PoCs with flash loan attacks or stale price exploitation
- List correct heartbeat intervals per feed
- Calculate impact of decimal mismatches

**DON'T:**
- Report missing features with alternative price sources
- Flag test/mock oracles in development
- Ignore chain-specific requirements (L2 sequencer)
- Miss multi-hop price calculations
