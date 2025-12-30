# Unfair Liquidation Security Checklist

Verify each item before finalizing audit report:

- [ ] **L2 sequencer grace period:** Grace period implemented after sequencer restart before liquidations enabled
- [ ] **Interest during pause:** Interest accrual paused when repayments paused, OR liquidation also paused
- [ ] **Synchronized pause states:** Repayment pause also pauses liquidation to prevent unfair liquidations
- [ ] **Fee updates before check:** All interest/fees accrued before `isLiquidatable()` evaluation
- [ ] **PNL/yield credit:** Positive unrealized PNL and earned yield credited during liquidation settlement
- [ ] **Health improvement:** Liquidation improves borrower health score, not just extracts value
- [ ] **Risk-based priority:** Collateral liquidated in order of risk (volatile before stable)
- [ ] **Position transfer handling:** Repayments routed correctly after position ownership transfer
- [ ] **LTV gap exists:** Gap between max borrow LTV and liquidation threshold (e.g., 80% borrow, 85% liquidate)
- [ ] **Auction interest pause:** Interest paused during liquidation auction period
- [ ] **Liquidator slippage:** Liquidation accepts minReward/maxDebt for slippage protection
