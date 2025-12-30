# Lending & Borrowing Security Checklist

Verify each item before finalizing audit report:

- [ ] **Liquidation timing:** Only possible after `paymentDueDate + gracePeriod`, not during cycle
- [ ] **Collateral integrity:** Records cannot be zeroed/overwritten after loan creation
- [ ] **Loan closure:** Requires full repayment verification (not just counter decrement)
- [ ] **Symmetric pause:** Repayment pause also pauses liquidations
- [ ] **Token restrictions:** Disallow only affects new loans, not existing repayments/liquidations
- [ ] **Grace period:** Exists after repayment resumption (unpause or token re-allow)
- [ ] **Liquidation shares:** Calculated from total debt, not single position
- [ ] **Repayment routing:** Sent to valid lender address (not zero/deleted)
- [ ] **Minimum loan size:** Enforced at creation and all modification points
- [ ] **Maximum loan ratio:** Validated on all loan operations (not just creation)
- [ ] **Interest precision:** Cannot result in zero due to rounding
- [ ] **Pool parameters:** Borrower can specify expected values to prevent front-running
- [ ] **Auction duration:** Has reasonable minimum (not 1 second auctions)
- [ ] **Atomic accounting:** Pool balance updates synchronized with loan state changes
- [ ] **Outstanding loans:** Tracked accurately (no double-counting or missed decrements)
