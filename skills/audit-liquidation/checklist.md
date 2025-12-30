# Liquidation Incentive Security Checklist

Verify each item before finalizing audit report:

- [ ] **Liquidation rewards:** Bonus/rewards implemented exceeding typical gas costs for trustless liquidators
- [ ] **Minimum position size:** Enforced to ensure all positions profitable to liquidate at typical gas prices
- [ ] **Collateral withdrawal restrictions:** Users cannot withdraw collateral while maintaining positions that could become underwater
- [ ] **Bad debt handling:** Insurance fund or socialization mechanism exists for insolvent positions
- [ ] **Partial liquidation support:** Enabled for positions exceeding reasonable liquidator capacity
- [ ] **Bad debt accounting:** Partial liquidations properly account for bad debt, prevent cherry-picking profitable portions
