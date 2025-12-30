# Concentrated Liquidity Manager Vulnerability Patterns

## Pattern #1: Forced Unfavorable Liquidity Deployment
**Risk:** Some functions deploy liquidity without TWAP checks, allowing MEV bots to sandwich attack and force protocol to deploy at manipulated prices
**Detection:** Verify ALL functions that mint positions or add liquidity validate current price against TWAP
**Impact:** Protocol loses funds to sandwich attacks, liquidity deployed at unfavorable prices causing immediate impermanent loss

## Pattern #2: Owner Rug-Pull via TWAP Parameters
**Risk:** Owner can set ineffective maxDeviation (e.g., 100%) or twapInterval (e.g., 1 second) that disable TWAP protection
**Detection:** Check if TWAP parameters have minimum/maximum bounds enforced
**Impact:** Owner disables protection and coordinates with MEV bot to sandwich attack protocol's liquidity deployments

## Pattern #3: Tokens Permanently Stuck
**Risk:** Rounding errors from Uniswap V3 position management accumulate tokens in contract that can never be withdrawn
**Detection:** Check if token balances can grow beyond what's in positions, and if sweep/rescue function exists
**Impact:** Protocol loses accumulated tokens permanently, can be significant over time

## Pattern #4: Stale Token Approvals
**Risk:** Updating router address doesn't revoke approvals to old router, allowing old compromised router to drain funds
**Detection:** Verify router updates revoke old approvals before setting new router
**Impact:** If old router compromised, attacker can drain all approved tokens

## Pattern #5: Retrospective Fee Application
**Risk:** Changing protocol fee percentage applies to already earned but uncollected rewards
**Detection:** Check if fees collected before fee structure updates
**Impact:** Users lose more fees than expected, protocol takes larger share of earned rewards retroactively
