# Staking & Reward Security Checklist

Verify each item before finalizing audit report:

- [ ] **Separate tokens:** Reward token cannot be same as staking token (prevents first depositor attack)
- [ ] **No direct transfer dilution:** totalSupply tracks staked amounts, not token balance
- [ ] **Precision protection:** Minimum stake enforced or sufficient scaling to prevent rounding to zero
- [ ] **Flash protection:** Time locks, minimum duration, or anti-sandwich mechanisms implemented
- [ ] **Index updates:** updateReward called before AND after reward distribution
- [ ] **Balance integrity:** Cached balances updated correctly during claims
