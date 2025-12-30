# Reentrancy Security Checklist

Verify each item before finalizing audit report:

- [ ] **CEI pattern:** State changes before external calls (Checks-Effects-Interactions)
- [ ] **NonReentrant modifiers:** Applied to all state-changing functions with external calls
- [ ] **Token assumptions:** No assumptions about token transfer behavior (assume callbacks possible)
- [ ] **Cross-function analysis:** Shared state variables protected across all functions
- [ ] **Read-only safety:** View functions return consistent values during reentrancy or document limitations
