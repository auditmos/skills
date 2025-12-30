# Oracle Integration Security Checklist

Verify each item before finalizing audit report:

- [ ] **Stale price checks:** updatedAt validated against appropriate heartbeat for each feed
- [ ] **L2 sequencer check:** Sequencer uptime verified on L2 deployments before using prices
- [ ] **Feed-specific heartbeats:** Each feed uses its documented heartbeat interval (not generic timeout)
- [ ] **Oracle precision:** decimals() method used, no hardcoded decimal assumptions
- [ ] **Price feed addresses:** Verified correct for specific chain and asset
- [ ] **Oracle revert handling:** latestRoundData() calls wrapped in try/catch with fallback
- [ ] **Depeg monitoring:** Wrapped assets have separate feeds or depeg detection (e.g., WBTC/BTC)
- [ ] **Min/max validation:** Prices checked against circuit breaker bounds (minAnswer/maxAnswer)
- [ ] **TWAP usage:** Time-weighted average used instead of spot prices where appropriate
- [ ] **Price direction:** Quote/base token order verified correct (not inverted)
- [ ] **Circuit breaker checks:** Returned price validated not exactly at min/max bounds
