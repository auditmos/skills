# Slippage Protection Vulnerabilities

1. **No Slippage Parameter** - Hard-coded 0 minimum output allows catastrophic MEV sandwich attacks
2. **No Expiration Deadline** - Transactions can be held and executed at unfavorable times
3. **Incorrect Slippage Calculation** - Using values other than minTokensOut for slippage protection
4. **Mismatched Slippage Precision** - Slippage not scaled to match output token decimals
5. **Hard-coded Slippage Freezes Funds** - Fixed slippage prevents withdrawals during high volatility
6. **MinTokensOut For Intermediate Amount** - Slippage only checked on intermediate, not final output
7. **On-Chain Slippage Calculation** - Using Quoter.quoteExactInput() subject to manipulation
8. **Fixed Fee Tier Assumption** - Hardcoding 3000 (0.3%) fee when pools may use different tiers
9. **Block.timestamp Deadline** - Using current timestamp provides no protection
