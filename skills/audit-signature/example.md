# Code Examples: Signature-Related Vulnerabilities

## Vulnerable Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VulnerableSignatureExamples {
    mapping(address => bool) public isWhitelisted;
    mapping(address => uint256) public balances;

    // Pattern #1: Missing Nonce Replay Protection
    function whitelistUser_VULNERABLE(
        address user,
        bytes memory signature
    ) external {
        // NO NONCE CHECK!
        // Signature can be replayed even after user is removed from whitelist
        bytes32 messageHash = keccak256(abi.encodePacked(user));
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);
        address signer = recoverSigner(ethSignedMessageHash, signature);

        require(signer == owner(), "Invalid signature");
        isWhitelisted[user] = true;
    }

    // Admin removes user, but signature can be replayed to re-whitelist!
    function removeWhitelist_VULNERABLE(address user) external {
        require(msg.sender == owner(), "Not owner");
        isWhitelisted[user] = false;
        // Old signature still valid - can be replayed!
    }

    // Pattern #2: Cross-Chain Replay (No chain_id)
    struct Permit {
        address owner;
        address spender;
        uint256 value;
        uint256 deadline;
        // MISSING: chainid
    }

    function permit_VULNERABLE(
        address owner_,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        // NO CHAIN_ID in hash!
        bytes32 structHash = keccak256(
            abi.encode(
                keccak256("Permit(address owner,address spender,uint256 value,uint256 deadline)"),
                owner_,
                spender,
                value,
                deadline
                // Missing: block.chainid
            )
        );

        bytes32 hash = keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash));
        address signer = ecrecover(hash, v, r, s);

        require(signer == owner_, "Invalid signature");
        // Signature from Mainnet can be replayed on Polygon, Arbitrum, etc!
        _approve(owner_, spender, value);
    }

    // Pattern #3: Missing Critical Parameters
    function withdraw_VULNERABLE(
        uint256 amount,
        address recipient,
        bytes memory signature
    ) external {
        // ONLY signs amount, NOT recipient!
        // Attacker can change recipient to their address
        bytes32 messageHash = keccak256(abi.encodePacked(amount));
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);
        address signer = recoverSigner(ethSignedMessageHash, signature);

        require(signer == owner(), "Invalid signature");
        // User signed to withdraw X tokens, but attacker changes recipient!
        balances[recipient] += amount;
    }

    function transfer_VULNERABLE(
        address from,
        address to,
        uint256 amount,
        bytes memory signature
    ) external {
        // MISSING: 'to' address not in signature!
        bytes32 messageHash = keccak256(abi.encodePacked(from, amount));
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);
        address signer = recoverSigner(ethSignedMessageHash, signature);

        require(signer == from, "Invalid signature");
        // Attacker can redirect transfer to any address!
        _transfer(from, to, amount);
    }

    // Pattern #4: No Expiration Deadline
    function executeWithSignature_VULNERABLE(
        address target,
        bytes calldata data,
        bytes memory signature
    ) external {
        // NO DEADLINE!
        // Signature valid forever - "lifetime license"
        bytes32 messageHash = keccak256(abi.encodePacked(target, data));
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);
        address signer = recoverSigner(ethSignedMessageHash, signature);

        require(signer == owner(), "Invalid signature");
        // Signature from 2020 still valid in 2030!
        // Old compromised signatures exploitable indefinitely
        (bool success, ) = target.call(data);
        require(success, "Call failed");
    }

    // Pattern #5: Unchecked ecrecover() Return
    function verifySignature_VULNERABLE(
        bytes32 messageHash,
        bytes memory signature
    ) public view returns (bool) {
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);
        address signer = recoverSigner(ethSignedMessageHash, signature);

        // NO CHECK: signer != address(0)!
        // If ecrecover fails, returns address(0)
        // If owner() == address(0) (uninitialized), this passes!
        return signer == owner();
    }

    function authorize_VULNERABLE(bytes32 hash, uint8 v, bytes32 r, bytes32 s) external {
        address signer = ecrecover(hash, v, r, s);
        // NO CHECK for address(0)!
        // Invalid signature returns address(0)
        // If checking against uninitialized variable (address(0)), passes!
        require(signer == expectedSigner, "Invalid signature");
        _grantAccess(signer);
    }

    // Pattern #6: Signature Malleability
    function claimReward_VULNERABLE(
        address user,
        uint256 amount,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        bytes32 messageHash = keccak256(abi.encodePacked(user, amount));
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);

        // Direct ecrecover without malleability protection
        address signer = ecrecover(ethSignedMessageHash, v, r, s);
        require(signer == trustedSigner, "Invalid signature");

        // NO PROTECTION against malleable signatures!
        // Given valid (v,r,s), attacker can compute (v',r,s') that also validates
        // Can be exploited if signature tracking uses hash(v,r,s) as unique ID
        require(!usedSignatures[keccak256(abi.encodePacked(v, r, s))], "Used");
        usedSignatures[keccak256(abi.encodePacked(v, r, s))] = true;

        // Attacker can claim twice with (v,r,s) and (v',r,s')
        balances[user] += amount;
    }

    // Helper functions (stubs)
    function owner() public view returns (address) { return address(this); }
    function recoverSigner(bytes32 ethSignedMessageHash, bytes memory signature) internal pure returns (address) {
        (bytes32 r, bytes32 s, uint8 v) = splitSignature(signature);
        return ecrecover(ethSignedMessageHash, v, r, s);
    }
    function splitSignature(bytes memory sig) internal pure returns (bytes32 r, bytes32 s, uint8 v) {
        require(sig.length == 65, "Invalid signature length");
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
    }
    function getEthSignedMessageHash(bytes32 messageHash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", messageHash));
    }
    function _approve(address owner_, address spender, uint256 value) internal {}
    function _transfer(address from, address to, uint256 amount) internal {}
    function _grantAccess(address user) internal {}

    bytes32 public DOMAIN_SEPARATOR;
    address public expectedSigner;
    address public trustedSigner;
    mapping(bytes32 => bool) public usedSignatures;
}
```

## Fixed Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {EIP712} from "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract FixedSignatureExamples is EIP712 {
    using ECDSA for bytes32;

    mapping(address => bool) public isWhitelisted;
    mapping(address => uint256) public balances;
    mapping(address => uint256) public nonces; // Nonce tracking

    constructor() EIP712("FixedSignatureExamples", "1") {}

    // Pattern #1 FIXED: Add Nonce Replay Protection
    function whitelistUser_FIXED(
        address user,
        uint256 nonce,
        bytes memory signature
    ) external {
        // VALIDATE NONCE!
        require(nonce == nonces[user], "Invalid nonce");

        bytes32 messageHash = keccak256(abi.encodePacked(user, nonce));
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
        address signer = ethSignedMessageHash.recover(signature);

        require(signer == owner(), "Invalid signature");

        // INCREMENT NONCE - signature can't be replayed
        nonces[user]++;

        isWhitelisted[user] = true;
    }

    function removeWhitelist_FIXED(address user) external {
        require(msg.sender == owner(), "Not owner");
        isWhitelisted[user] = false;

        // Increment nonce to invalidate any pending signatures
        nonces[user]++;
    }

    // Pattern #2 FIXED: Include chain_id (Using OpenZeppelin EIP712)
    bytes32 public constant PERMIT_TYPEHASH = keccak256(
        "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
    );

    function permit_FIXED(
        address owner_,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        require(block.timestamp <= deadline, "Signature expired");

        // EIP712 includes chain_id in domain separator automatically
        bytes32 structHash = keccak256(
            abi.encode(
                PERMIT_TYPEHASH,
                owner_,
                spender,
                value,
                nonces[owner_],
                deadline
            )
        );

        bytes32 hash = _hashTypedDataV4(structHash);
        address signer = hash.recover(v, r, s);

        require(signer == owner_, "Invalid signature");
        require(signer != address(0), "Invalid signer");

        nonces[owner_]++;
        _approve(owner_, spender, value);
    }

    // Pattern #3 FIXED: Include All Critical Parameters
    function withdraw_FIXED(
        uint256 amount,
        address recipient,
        uint256 nonce,
        uint256 deadline,
        bytes memory signature
    ) external {
        require(block.timestamp <= deadline, "Signature expired");
        require(nonce == nonces[owner()], "Invalid nonce");

        // INCLUDE ALL PARAMETERS: amount AND recipient
        bytes32 messageHash = keccak256(
            abi.encodePacked(amount, recipient, nonce, deadline)
        );
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
        address signer = ethSignedMessageHash.recover(signature);

        require(signer == owner(), "Invalid signature");
        require(signer != address(0), "Invalid signer");

        nonces[owner()]++;
        balances[recipient] += amount;
    }

    bytes32 public constant TRANSFER_TYPEHASH = keccak256(
        "Transfer(address from,address to,uint256 amount,uint256 nonce,uint256 deadline)"
    );

    function transfer_FIXED(
        address from,
        address to,
        uint256 amount,
        uint256 nonce,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        require(block.timestamp <= deadline, "Signature expired");
        require(nonce == nonces[from], "Invalid nonce");

        // ALL PARAMETERS in struct hash
        bytes32 structHash = keccak256(
            abi.encode(
                TRANSFER_TYPEHASH,
                from,
                to,
                amount,
                nonce,
                deadline
            )
        );

        bytes32 hash = _hashTypedDataV4(structHash);
        address signer = hash.recover(v, r, s);

        require(signer == from, "Invalid signature");
        require(signer != address(0), "Invalid signer");

        nonces[from]++;
        _transfer(from, to, amount);
    }

    // Pattern #4 FIXED: Add Expiration Deadline
    bytes32 public constant EXECUTE_TYPEHASH = keccak256(
        "Execute(address target,bytes data,uint256 nonce,uint256 deadline)"
    );

    function executeWithSignature_FIXED(
        address target,
        bytes calldata data,
        uint256 nonce,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        // VALIDATE DEADLINE!
        require(block.timestamp <= deadline, "Signature expired");
        require(nonce == nonces[owner()], "Invalid nonce");

        bytes32 structHash = keccak256(
            abi.encode(
                EXECUTE_TYPEHASH,
                target,
                keccak256(data),
                nonce,
                deadline
            )
        );

        bytes32 hash = _hashTypedDataV4(structHash);
        address signer = hash.recover(v, r, s);

        require(signer == owner(), "Invalid signature");
        require(signer != address(0), "Invalid signer");

        nonces[owner()]++;

        (bool success, ) = target.call(data);
        require(success, "Call failed");
    }

    // Pattern #5 FIXED: Check ecrecover() Return for address(0)
    function verifySignature_FIXED(
        bytes32 messageHash,
        bytes memory signature
    ) public view returns (bool) {
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
        address signer = ethSignedMessageHash.recover(signature);

        // CHECK: signer != address(0)!
        require(signer != address(0), "Invalid signature");

        return signer == owner();
    }

    function authorize_FIXED(bytes32 hash, uint8 v, bytes32 r, bytes32 s) external {
        address signer = ecrecover(hash, v, r, s);

        // VALIDATE: not address(0)
        require(signer != address(0), "Invalid signature");
        require(signer == expectedSigner, "Invalid signer");

        _grantAccess(signer);
    }

    // Pattern #6 FIXED: Use OpenZeppelin ECDSA to Prevent Malleability
    bytes32 public constant CLAIM_TYPEHASH = keccak256(
        "Claim(address user,uint256 amount,uint256 nonce)"
    );

    function claimReward_FIXED(
        address user,
        uint256 amount,
        uint256 nonce,
        bytes memory signature
    ) external {
        require(nonce == nonces[user], "Invalid nonce");

        bytes32 structHash = keccak256(
            abi.encode(CLAIM_TYPEHASH, user, amount, nonce)
        );

        bytes32 hash = _hashTypedDataV4(structHash);

        // OpenZeppelin ECDSA.recover includes malleability protection
        // Rejects signatures with s > secp256k1n/2
        address signer = hash.recover(signature);

        require(signer == trustedSigner, "Invalid signature");
        require(signer != address(0), "Invalid signer");

        // Nonce-based replay protection (better than signature hash tracking)
        nonces[user]++;

        balances[user] += amount;
    }

    // Alternative: Manual malleability check
    function claimReward_FIXED_MANUAL(
        address user,
        uint256 amount,
        uint256 nonce,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        require(nonce == nonces[user], "Invalid nonce");

        // MALLEABILITY CHECK: Reject high-s values
        // secp256k1 curve order n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
        require(
            uint256(s) <= 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0,
            "Invalid signature 's' value"
        );
        // Also ensure v is 27 or 28
        require(v == 27 || v == 28, "Invalid signature 'v' value");

        bytes32 structHash = keccak256(
            abi.encode(CLAIM_TYPEHASH, user, amount, nonce)
        );

        bytes32 hash = _hashTypedDataV4(structHash);
        address signer = ecrecover(hash, v, r, s);

        require(signer == trustedSigner, "Invalid signature");
        require(signer != address(0), "Invalid signer");

        nonces[user]++;
        balances[user] += amount;
    }

    // Helper functions (stubs)
    function owner() public view returns (address) { return address(this); }
    function _approve(address owner_, address spender, uint256 value) internal {}
    function _transfer(address from, address to, uint256 amount) internal {}
    function _grantAccess(address user) internal {}

    address public expectedSigner;
    address public trustedSigner;
}
```
