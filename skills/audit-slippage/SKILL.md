---
name: audit-slippage
description: Audits Solidity DEX integrations and smart contracts for slippage vulnerabilities enabling sandwich attacks including zero/missing minAmountOut parameters, block.timestamp or missing deadlines, on-chain slippage calculation via manipulable quoters, multi-hop swaps without final output protection, decimal precision mismatches between token pairs, hard-coded slippage preventing withdrawals during volatility, and fixed fee tier assumptions breaking when liquidity migrates
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Slippage Protection Auditor

## When to Use
- Auditing DEX integrations, AMM interactions, swap operations
- User mentions: slippage, MEV, sandwich attack, front-running, deadline, minAmountOut, swap, Uniswap, Curve, Balancer
- Reviewing token exchange functions, liquidity operations, router integrations
- Analyzing price impact protection in DeFi protocols

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-slippage** skill to analyze this contract for slippage protection and MEV vulnerabilities..."

1. **Identify swap/liquidity operations**
   - Search: `swap`, `addLiquidity`, `removeLiquidity`, `IUniswapV2Router`, `ISwapRouter`
   - Focus: minAmountOut parameters, deadline parameters, quoter usage

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate MEV exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit via MEV?
   - Can MEV bot sandwich attack?
   - Calculate extractable value (% of trade)
   - Check if protection exists elsewhere in call stack
   - Downgrade severity if admin-only unless users affected by MEV

4. **Generate report**
   - Use deliverable template below
   - Include sandwich attack PoC
   - Quantify MEV extraction potential

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. No slippage parameter (minAmountOut = 0) → 99%+ value extractable
2. No expiration deadline (type(uint256).max) → delayed execution risk
3. block.timestamp as deadline → zero protection
4. Incorrect slippage calculation → wrong reference value
5. Mismatched slippage precision → decimal scaling errors
6. Hard-coded slippage → withdrawal failures during volatility
7. MinTokensOut for intermediate amount → multi-hop unprotected
8. On-chain slippage calculation → flash loan manipulation
9. Fixed fee tier assumption → routing through wrong pool
10. Slippage on token amount not USD value → market crash risk
11. No slippage on liquidity ops → LP value extraction
12. Flash swap repayment without slippage → overpayment risk
13. Approval race on router upgrade → MEV via old router

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Zero slippage on user-facing swaps, missing deadline, on-chain quoter-based minOut, **MUST be exploitable by non-privileged actors**
**High:** Hard-coded slippage preventing withdrawals, intermediate-hop-only protection, wrong fee tier (80%+ liquidity elsewhere), **MUST be exploitable by non-privileged actors**
**Medium:** Wrong decimal precision (user can retry), suboptimal routing, missing LP operation slippage, **admin-only swap functions with cascading user MEV exposure**
**Low:** Suboptimal slippage (token vs USD) in stable pairs, documentation issues, **admin-only swap parameter issues without immediate user impact**

**IMPORTANT:** Admin-only swap functions (onlyOwner, onlyAdmin, onlyGovernance) are **MEDIUM or LOW severity** unless:
- Admin swaps use user funds directly (e.g., fee collection selling user-deposited tokens)
- Missing slippage enables admin to extract value from protocol treasury holding user funds
- Admin swap parameters affect user swap routing or slippage calculations

## False Positives - Do NOT Flag

- Zero slippage on internal protocol-to-protocol swaps (both sides controlled)
- block.timestamp deadline in keeper/bot functions with off-chain slippage enforcement
- Hard-coded slippage in emergency-only functions with explicit warnings
- On-chain quoter in view functions (display/estimation only)
- Fixed fee tier with documented single-pool targeting
- **Admin-only swap functions** (onlyOwner, onlyAdmin) swapping protocol-owned assets not derived from user funds
- Governance-controlled swaps with timelock allowing users to exit before execution
- Treasury management swaps where admin has no access to user deposits

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, impact (MEV extraction %), PoC with sandwich attack simulation, remediation, gas impact.

## Key Principles

- **User control** - users specify slippage and deadline per tx
- **Off-chain calculation** - minAmountOut from off-chain or TWAP, never current block
- **Final output protection** - multi-hop must protect final amount, not intermediate
- **Decimal awareness** - account for token decimal differences

## Output Guidelines

**DO:**
- Reference specific lines/functions
- Provide sandwich attack PoCs
- Quantify MEV extraction ($ or %)
- Include real-world exploit examples

**DON'T:**
- Flag view/pure functions (no state change)
- Report intentional designs without exploit path
- Use vague terms
- Ignore liquidity depth context
