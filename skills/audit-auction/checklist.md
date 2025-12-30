# Auction Manipulation Security Checklist

Verify each item before finalizing audit report:

- [ ] **No self-bidding:** Borrower/owner cannot bid on own auction or reset timer
- [ ] **Sequencer check:** Sequencer uptime validated before auction start on L2 chains
- [ ] **Minimum length:** Auction length has enforced minimum (e.g., 1 hour)
- [ ] **Correct timestamp comparison:** Auction end check uses >= not > (no off-by-one)
