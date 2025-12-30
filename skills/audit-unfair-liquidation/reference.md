# Unfair Liquidation Vulnerability Patterns

## Pattern #1: Missing L2 Sequencer Grace Period
**Risk:** Users on L2 chains (Arbitrum, Optimism) liquidated immediately when sequencer comes back online, with no time to respond to price changes during downtime
**Detection:** Check for sequencer uptime oracle integration and grace period after sequencer restart
**Impact:** Mass unfair liquidations after sequencer downtime, users unable to add collateral or repay

## Pattern #2: Interest Accumulates While Paused
**Risk:** Interest/fees continue accruing while protocol paused, causing users to become liquidatable without ability to repay during pause
**Detection:** Verify interest accrual stops when repayment paused, or liquidation also paused
**Impact:** Users liquidated for debt they couldn't have repaid during downtime

## Pattern #3: Repayment Paused, Liquidation Active
**Risk:** Protocol pauses repayments (e.g., for upgrade) but liquidation remains active, preventing users from avoiding liquidation
**Detection:** Check if repayment pause state also disables liquidation
**Impact:** Guaranteed unfair liquidations - users cannot defend positions

## Pattern #4: Late Interest/Fee Updates
**Risk:** `isLiquidatable()` check uses stale interest values, not calling `accrueInterest()` first
**Detection:** Verify all fee/interest accumulators updated before liquidation eligibility check
**Impact:** Users liquidated based on stale data, or liquidations fail when actual values differ

## Pattern #5: Lost Positive PNL/Yield
**Risk:** Profitable positions with unrealized gains or earned yield lose these during liquidation
**Detection:** Check if positive PNL/yield credited to borrower during liquidation settlement
**Impact:** Users lose earned profits, liquidation takes more than necessary

## Pattern #6: Unhealthier Post-Liquidation State
**Risk:** Liquidator cherry-picks stable collateral (USDC, WETH), leaving borrower with only volatile assets
**Detection:** Verify liquidation improves health score or follows risk-based collateral priority
**Impact:** Users left with worse risk profile, cascading liquidations more likely

## Pattern #7: Corrupted Collateral Priority
**Risk:** Liquidation order doesn't match risk profile - volatile assets should be liquidated first
**Detection:** Check if collateral liquidation follows risk-weighted priority (volatile before stable)
**Impact:** Protocol accumulates higher-risk collateral, increases systemic risk

## Pattern #8: Borrower Replacement Misattribution
**Risk:** After position transfer, original borrower's repayments credited to new owner
**Detection:** Check repayment routing when positions are transferable
**Impact:** Original borrower loses funds, new owner gets free repayment

## Pattern #9: No LTV Gap
**Risk:** Borrow LTV equals liquidation LTV, so positions can be liquidated immediately after borrowing on any price movement
**Detection:** Verify gap between maximum borrow LTV and liquidation threshold
**Impact:** Users liquidated before having opportunity to manage position

## Pattern #10: Interest During Auction
**Risk:** Borrowers continue accruing interest while their position is being auctioned for liquidation
**Detection:** Check if interest pauses during auction period
**Impact:** Auction proceeds may not cover debt if interest continues growing

## Pattern #11: No Liquidation Slippage Protection
**Risk:** Liquidators cannot specify minimum acceptable rewards, MEV can sandwich liquidation transactions
**Detection:** Check if liquidation functions accept minReward/maxDebt parameters
**Impact:** Liquidators receive less than expected, reduces liquidation reliability
