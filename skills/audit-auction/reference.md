# Auction Manipulation Vulnerability Patterns

## Pattern #1: Self-Bidding to Reset Auction
**Risk:** Borrower can bid on own loan/auction to reset auction timer, extending indefinitely to avoid liquidation
**Detection:** Check if auction allows bidder to be borrower, or if winning bid resets timer
**Impact:** Borrower avoids liquidation indefinitely by repeatedly self-bidding, protocol accumulates bad debt

## Pattern #2: Auction Start During Sequencer Downtime
**Risk:** On L2 chains, auctions starting during sequencer downtime give unfair advantage once sequencer restarts
**Detection:** Verify sequencer uptime checked before auction start on L2 deployments
**Impact:** Unfair auction conditions, first bidder after restart has monopoly during catch-up period

## Pattern #3: Insufficient Auction Length Validation
**Risk:** No minimum auction length allows creating 1-second auctions for immediate seizure bypassing competitive bidding
**Detection:** Check if auction length has minimum bound (e.g., 1 hour)
**Impact:** Liquidator can seize collateral immediately without competitive bidding, borrower gets unfair price

## Pattern #4: Auction Can Be Seized During Active Period
**Risk:** Off-by-one error in timestamp check (using > instead of >=) allows seizure at exact auction end time
**Detection:** Verify auction end check uses >= not >, ensuring grace period respected
**Impact:** Liquidator seizes collateral during auction period before bidders can respond
