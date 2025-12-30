# Oracle Integration Vulnerability Patterns

## Pattern #1: Not Checking Stale Prices
**Risk:** Missing updatedAt validation against heartbeat intervals allows using outdated prices during market volatility or oracle failures
**Detection:** Verify updatedAt from latestRoundData() compared against feed-specific heartbeat (not generic timeout)
**Impact:** Protocol uses stale prices for liquidations, collateral valuation, swaps - enabling arbitrage and unfair liquidations

## Pattern #2: Missing L2 Sequencer Check
**Risk:** L2 chains (Arbitrum, Optimism) require sequencer uptime validation - prices during downtime are stale
**Detection:** Check for sequencer uptime feed integration on L2 deployments
**Impact:** Stale prices used after sequencer restart, mass unfair liquidations, price manipulation during downtime

## Pattern #3: Same Heartbeat For Multiple Feeds
**Risk:** Different Chainlink feeds have different heartbeat intervals (BTC: 1h, ETH: 1h, stablecoins: 24h)
**Detection:** Verify each feed uses its specific documented heartbeat, not hardcoded single value
**Impact:** False staleness rejections or accepting actually stale prices

## Pattern #4: Assuming Oracle Precision
**Risk:** Different feeds return different decimals (most: 8, some: 18) - hardcoding decimals causes errors
**Detection:** Check if code calls decimals() method or hardcodes decimal assumption
**Impact:** Prices off by 10^10, catastrophic miscalculation of collateral/debt values

## Pattern #5: Incorrect Price Feed Address
**Risk:** Using wrong feed address (mainnet address on testnet, ETH/USD instead of WETH/USD)
**Detection:** Verify feed addresses match Chainlink documentation for specific chain and asset
**Impact:** Completely wrong prices, protocol insolvency

## Pattern #6: Unhandled Oracle Reverts
**Risk:** Oracle calls not wrapped in try/catch - any oracle failure causes complete protocol DoS
**Detection:** Check if latestRoundData() wrapped in error handling
**Impact:** Protocol becomes unusable during oracle maintenance or failures

## Pattern #7: Unhandled Depeg Events
**Risk:** Using BTC/USD for WBTC ignores bridge compromise scenarios where WBTC depegs from BTC
**Detection:** Verify wrapped assets have separate feeds or depeg monitoring (WBTC/BTC feed)
**Impact:** Catastrophic losses when wrapped asset loses peg, collateral becomes worthless while oracle shows full value

## Pattern #8: Oracle Min/Max Price Issues
**Risk:** Chainlink has minAnswer/maxAnswer bounds - during flash crashes oracle returns bound value, not actual price
**Detection:** Check if code validates price isn't exactly at circuit breaker bounds
**Impact:** Protocol uses incorrect price during extreme volatility, enables liquidation manipulation

## Pattern #9: Using Slot0 Price
**Risk:** Uniswap V3 slot0 price is spot price, manipulable via flash loans in single transaction
**Detection:** Check if code reads slot0 directly without TWAP
**Impact:** Flash loan price manipulation, steal funds via manipulated collateral valuation

## Pattern #10: Price Feed Direction Confusion
**Risk:** Using opposite token pair direction (DAI/USD when needing USD/DAI) results in inverted pricing
**Detection:** Verify quote/base token order matches protocol needs
**Impact:** Inverted prices, collateral overvalued or undervalued by square of error

## Pattern #11: Missing Circuit Breaker Checks
**Risk:** Not checking if returned price equals minAnswer/maxAnswer bounds during extreme events
**Detection:** Verify code checks price != minAnswer && price != maxAnswer
**Impact:** Protocol uses circuit breaker bound values as real prices during flash crashes
