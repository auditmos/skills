# Concentrated Liquidity Manager Security Checklist

Verify each item before finalizing audit report:

- [ ] **TWAP checks everywhere:** ALL functions deploying liquidity validate current price against TWAP
- [ ] **TWAP parameter bounds:** maxDeviation and twapInterval have enforced min/max limits
- [ ] **No token accumulation:** No tokens stuck in contract beyond active positions, or sweep function exists
- [ ] **Approval revocation:** Old approvals revoked before router updates
- [ ] **Fee immutability:** Fees collected before fee structure changes
