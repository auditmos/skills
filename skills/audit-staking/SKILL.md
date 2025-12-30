---
name: audit-staking
description: Audits Solidity staking and reward protocols for vulnerabilities including front-running first deposit to steal initial rewards, reward dilution via direct transfers, precision loss in reward calculations causing rounding to zero, flash deposit/withdraw griefing diluting rewards, update not called after reward distribution causing stale index, and balance caching issues during claims (project)
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Staking & Reward Auditor

## When to Use
- Auditing staking mechanisms, reward distribution, yield farming
- User mentions: staking, rewards, yield, farming, rewardPerToken, deposit, withdraw, claim, first depositor
- Analyzing reward calculations, index updates, share dilution
- Reviewing deposit/withdraw flows, precision handling

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-staking** skill to analyze this contract for staking and reward vulnerabilities..."

1. **Scan for staking operations**
   - Search: `stake`, `deposit`, `withdraw`, `claim`, `rewardPerToken`, `totalSupply`, `balanceOf`, `earned`, `updateReward`
   - Focus: reward calculations, first depositor, direct transfers, precision loss, flash actions

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit? (first depositor steal, direct transfer dilution, flash griefing)
   - Can direct transfers dilute rewards?
   - Do small amounts round to zero?
   - Can flash deposits/withdraws grief stakers?
   - Is update called after distribution?
   - Are balances cached correctly?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless direct user impact

4. **Generate report**
   - Use deliverable template below
   - Include reward theft/dilution analysis and PoC
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Front-running first deposit → attacker steals initial WETH rewards via sandwich attack
2. Reward dilution via direct transfer → sending tokens directly increases totalSupply without staking
3. Precision loss in rewards → small stakes or frequent updates cause rewards rounding to zero
4. Flash deposit/withdraw griefing → large instant deposits dilute rewards for existing stakers
5. Update not called after distribution → stale index causes incorrect reward calculations
6. Balance caching issues → claiming updates cached balance incorrectly

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** First depositor can steal all initial rewards, direct transfer dilution enabling theft, flash deposit/withdraw draining rewards, **MUST be exploitable by non-privileged actors**
**High:** Precision loss causing rewards to round to zero for legitimate users, stale index after distribution, **MUST be exploitable by non-privileged actors**
**Medium:** Suboptimal reward distribution timing, griefing via large flash actions without theft, admin-only reward configuration issues with user impact
**Low:** Gas inefficiencies in reward calculations, missing events, admin-only parameter issues without immediate user impact

**IMPORTANT:** Admin-only reward functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Invalid reward parameters directly steal/brick user rewards
- Missing validation enables admin rug pull of staking pool
- Error cascades to all users immediately (e.g., division by zero in reward calculation)

## False Positives - Do NOT Flag

- Protocols requiring minimum stake amounts (prevents dust)
- Reward tokens same as staking tokens by design
- Intentional admin-only initial deposit
- View functions with documented staleness
- Protocols with explicit front-running protection

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, reward theft analysis, PoC, remediation.

## Key Principles

- **Separate tokens** - reward token must differ from staking token
- **No direct transfers** - track staked amounts separately from balances
- **Precision protection** - minimum stake, scale factors for small amounts
- **Index updates** - call updateReward before and after distribution
- **Flash protection** - time locks or minimum stake duration
- **Balance integrity** - careful caching during claims

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide reward theft scenarios with calculations
- Show PoCs demonstrating reward extraction
- Calculate precision loss magnitude
- Map update call timing

**DON'T:**
- Report intentional design choices (same token staking)
- Flag missing features with alternative mechanisms
- Ignore precision implications (critical for accuracy)
- Miss edge cases (first deposit, zero stakes)
