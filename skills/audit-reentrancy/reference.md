# Reentrancy Vulnerability Patterns

## Pattern #1: Token Transfer Reentrancy
**Risk:** ERC777, ERC721, or tokens with callback hooks allow reentrancy during token transfers
**Detection:** Check if token transfers occur before state updates, or if nonReentrant modifier is missing
**Impact:** Attacker can reenter during token transfer callback to manipulate state or drain funds

## Pattern #2: State Update After External Call
**Risk:** State variables updated after external calls (transfer, call, etc.) violate CEI pattern
**Detection:** Verify all state changes occur before external calls in function execution flow
**Impact:** Attacker reenters between external call and state update to drain funds via repeated withdrawals

## Pattern #3: Cross-Function Reentrancy
**Risk:** Function A makes external call, attacker reenters Function B to manipulate shared state
**Detection:** Map all external calls and check if other functions can be called that access same state variables
**Impact:** Bypass single-function reentrancy guards by reentering different function that shares state

## Pattern #4: Read-Only Reentrancy
**Risk:** View/pure functions read state during reentrancy callback, return stale values used for critical decisions
**Detection:** Check if external protocols rely on view functions that could return inconsistent state during reentrancy
**Impact:** Attacker exploits stale state reads to manipulate prices, collateral ratios, or other derived values
