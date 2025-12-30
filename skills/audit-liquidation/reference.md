# Liquidation Incentive Vulnerability Patterns

## Pattern #1: No Liquidation Incentive
**Risk:** Trustless liquidators have no economic incentive (rewards/bonuses) to perform liquidations when gas costs exceed profits
**Detection:** Check if liquidation provides bonus collateral, rewards, or fees exceeding gas costs
**Impact:** Underwater positions remain unliquidated, protocol accumulates bad debt

## Pattern #2: No Incentive To Liquidate Small Positions
**Risk:** Small positions below gas cost threshold are unprofitable to liquidate, accumulate as bad debt
**Detection:** Verify minimum position sizes enforced to ensure liquidation profitability
**Impact:** Dust positions become protocol liabilities, systemic insolvency risk

## Pattern #3: Profitable User Withdraws All Collateral
**Risk:** Users with positive PNL can withdraw collateral while maintaining positions, removing liquidation incentive if position becomes underwater later
**Detection:** Check if collateral withdrawals are restricted based on position health/margin requirements
**Impact:** Positions become unliquidatable, guaranteed bad debt on negative price movements

## Pattern #4: No Mechanism To Handle Bad Debt
**Risk:** Insolvent positions (debt > collateral) have no recovery mechanism - no insurance fund or socialization
**Detection:** Verify insurance fund exists or bad debt socialization mechanism implemented
**Impact:** Protocol absorbs losses directly, becomes insolvent

## Pattern #5: Partial Liquidation Bypasses Bad Debt Accounting
**Risk:** Liquidators can partially liquidate profitable portion, leaving bad debt with protocol
**Detection:** Ensure partial liquidation properly accounts for bad debt and requires full position closure when insolvent
**Impact:** Protocol subsidizes liquidators, accumulates bad debt

## Pattern #6: No Partial Liquidation Prevents Whale Liquidation
**Risk:** Large positions exceed individual liquidator capital capacity, cannot be liquidated in single transaction
**Detection:** Verify partial liquidation supported for positions exceeding reasonable liquidator capacity
**Impact:** Whale positions remain underwater, accumulate as systemic bad debt
