---
name: audit-signature
description: Audits Solidity smart contracts for signature-related vulnerabilities including missing nonce replay protection allowing signatures to be reused after state changes, cross-chain replay attacks without chain_id validation, missing critical parameters in signed messages enabling manipulation, signatures without expiration timestamps granting lifetime access, unchecked ecrecover return values allowing invalid signatures, and signature malleability via elliptic curve symmetry
allowed-tools: Read, Grep, Glob
license: MIT
compatibility: Designed for Claude Code (or similar products)
metadata:
  author: Tomasz Kowalczyk (tom@auditmos.com)
  version: "1.0"
---

# Signature Security Auditor

## When to Use
- Auditing Solidity contracts using EIP-712 signatures, permits, meta-transactions
- User mentions: signature, ecrecover, EIP-712, permit, nonce, chain_id, deadline, replay, malleability
- Analyzing gasless transactions, delegation systems, authorization mechanisms
- Reviewing contracts with off-chain signing and on-chain verification

## Audit Workflow

**IMPORTANT: Announce skill usage at the start of analysis**

Begin with: "I'm using the **audit-signature** skill to analyze this contract for signature security vulnerabilities..."

1. **Scan for signature operations**
   - Search: `ecrecover`, `_hashTypedDataV4`, `permit`, `EIP712`, `signature`, `v, r, s` parameters
   - Focus: signature verification functions, nonce management, parameter inclusion

2. **Check against vulnerability patterns**
   - Reference `reference.md` for complete checklist
   - Compare code against `example.md`

3. **Validate exploitability**
   - **Check access control first** - grep for `onlyOwner|onlyAdmin|onlyGovernance` modifiers
   - Can non-privileged actors exploit signature issues?
   - Can signature be replayed after revocation?
   - Can signature be used on different chains?
   - Are all critical parameters signed?
   - Verify no compensating protections exist
   - Downgrade severity if admin-only unless enables governance attack

4. **Generate report**
   - Use deliverable template below
   - Include PoC for each finding
   - Rank by severity

## Core Vulnerability Patterns

See `reference.md` for full checklist. Key patterns:

1. Missing nonce replay protection → signatures reused after state changes
2. Cross-chain replay (no chain_id) → same signature valid on all chains
3. Missing critical parameters → unsigned params manipulated by attacker
4. No expiration deadline → lifetime signatures exploitable indefinitely
5. Unchecked ecrecover() return → address(0) passes validation
6. Signature malleability → alternative valid signatures computed without private key

**Code examples:** See `example.md`

## Severity Criteria

**Critical:** Missing nonce allowing replay after revocation, unchecked ecrecover() return allowing address(0) to authorize, cross-chain replay enabling unauthorized access on different chains, **MUST be exploitable by non-privileged actors**
**High:** Missing expiration allowing long-term exploitation, critical parameters (amount, recipient) not included in signature, **MUST be exploitable by non-privileged actors**
**Medium:** Signature malleability without OpenZeppelin ECDSA, missing parameters in low-impact contexts, admin signature functions with validation issues
**Low:** Nonce implementation suboptimal but functional, documentation issues, admin-only signature issues without user impact

**IMPORTANT:** Admin-only signature verification functions (onlyOwner, onlyAdmin) are **LOW severity** unless:
- Missing validation enables unauthorized user to forge admin signatures
- Signature replay/cross-chain issues allow governance takeover
- Unchecked ecrecover in admin function can be exploited via user-facing entry point

## False Positives - Do NOT Flag

- OpenZeppelin EIP712 and ECDSA libraries (already secure)
- Signatures with explicit "no expiration by design" documentation for specific use cases
- Internal signature verification with additional access controls
- Test/mock contracts with simplified signature schemes
- Nonces tracked via mapping(address => uint256) with proper increment (valid pattern)

## Deliverable Format

**MANDATORY:** Before deliverable, verify each `checklist.md` item against codebase. Flag violations as findings.

Use template: `templates/report-template.md`

Each finding includes: severity, pattern #, file/lines, description, vulnerable code, impact (replay attack, unauthorized access), PoC showing signature reuse or manipulation, remediation, gas impact.

## Key Principles

- **Nonce uniqueness** - every signature consumed exactly once
- **Chain-specific** - include chain_id in EIP-712 domain
- **Complete signing** - all relevant parameters in signed message
- **Time-bounded** - expiration deadlines for all signatures
- **Validated recovery** - check ecrecover() != address(0)
- **Malleability protection** - use OpenZeppelin ECDSA library

## Output Guidelines

**DO:**
- Reference specific lines and functions
- Provide executable replay attack PoCs
- Quantify impact (funds at risk, unauthorized operations)
- Show how to compute alternative signatures (malleability)

**DON'T:**
- Flag OpenZeppelin implementations without modifications
- Report intentional lifetime permits with clear documentation
- Use vague terms ("might be vulnerable")
- Ignore context (additional access controls, trusted signers)
