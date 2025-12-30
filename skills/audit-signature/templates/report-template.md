# Signature Security Audit Report

**Contract:** [Contract Name]
**Files Analyzed:** [List of .sol files]
**Vulnerabilities Found:** Critical: X | High: Y | Medium: Z | Low: W

---

## [SEVERITY] Vulnerability Title
**Pattern:** #[1-6]
**File:** `path/to/Contract.sol`
**Lines:** [line numbers]
**Function:** `functionName()`

### Description
[2-3 sentences explaining the signature vulnerability and its security impact]

### Vulnerable Code
```solidity
// Actual code from contract with line numbers
45: function whitelistUser(address user, bytes memory signature) external {
46:     // NO NONCE - signature can be replayed!
47:     bytes32 messageHash = keccak256(abi.encodePacked(user));
48:     address signer = recoverSigner(messageHash, signature);
49:     require(signer == owner, "Invalid signature");
50:     isWhitelisted[user] = true;
51: }
```

### Impact
- **Attack Vector:** [Replay attack / Cross-chain replay / Parameter manipulation / Signature forgery]
- **Exploitability:** [High/Medium/Low] - [Explanation of attack requirements]
- **Impact Severity:** [Unauthorized access / Fund theft / Privilege escalation / Indefinite access]
- **Affected Functions:** [List if multiple functions share this pattern]
- **Funds at Risk:** [Amount or percentage if quantifiable]

### Proof of Concept
```solidity
function testReplayAttack() public {
    // Step 1: User gets whitelisted with valid signature
    bytes memory validSignature = getOwnerSignature(user);
    contract.whitelistUser(user, validSignature);
    assert(contract.isWhitelisted(user) == true);

    // Step 2: Admin removes user from whitelist
    vm.prank(owner);
    contract.removeWhitelist(user);
    assert(contract.isWhitelisted(user) == false);

    // Step 3: Attacker replays old signature to re-whitelist
    contract.whitelistUser(user, validSignature); // Same signature!

    // Result: User re-whitelisted despite removal
    assert(contract.isWhitelisted(user) == true);
    // Attacker can replay indefinitely after each removal
}
```

### Remediation
```solidity
// Fixed code with nonce tracking
mapping(address => uint256) public nonces;

function whitelistUser(
    address user,
    uint256 nonce,
    bytes memory signature
) external {
    // Validate nonce
    require(nonce == nonces[user], "Invalid nonce");

    bytes32 messageHash = keccak256(abi.encodePacked(user, nonce));
    bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
    address signer = ethSignedMessageHash.recover(signature);

    require(signer == owner, "Invalid signature");

    // Increment nonce - prevents replay
    nonces[user]++;
    isWhitelisted[user] = true;
}

function removeWhitelist(address user) external {
    require(msg.sender == owner, "Not owner");
    isWhitelisted[user] = false;
    // Increment nonce to invalidate pending signatures
    nonces[user]++;
}

// Or use OpenZeppelin EIP712 for standardized implementation
import {EIP712} from "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
```

**Gas Impact:** +[X] gas per call (nonce storage/increment), negligible vs security improvement

---

## Summary

### Critical Issues Requiring Immediate Attention
1. [Issue #X] - [Missing nonce replay protection] - [Signatures reusable after revocation, unlimited privilege escalation]
2. [Issue #Y] - [Unchecked ecrecover return] - [address(0) bypasses authorization, complete protocol takeover]

### Recommendations
- Implement nonce-based replay protection for all signature operations
- Include chain_id in EIP-712 domain separator (use OpenZeppelin EIP712)
- Include all critical parameters (amount, recipient, target) in signed messages
- Add expiration deadlines to all signatures (typically 15 minutes to 1 hour)
- Always validate ecrecover() return != address(0)
- Use OpenZeppelin ECDSA library to prevent signature malleability
- Follow Checks-Effects-Interactions (CEI) pattern: verify signature before state changes
- Consider using EIP-2612 permit() pattern for token approvals
- Increment nonces on revocation/state changes to invalidate pending signatures
- Document signature schemes and security assumptions in natspec comments
