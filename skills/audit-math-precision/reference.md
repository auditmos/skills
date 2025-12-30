# Precision & Mathematical Vulnerabilities

1. **Division Before Multiplication** - Always multiply before dividing to minimize rounding errors
2. **Rounding Down To Zero** - Small values can round to 0, allowing state changes without proper accounting
3. **No Precision Scaling** - Mixing tokens with different decimals without scaling causes calculation errors
4. **Excessive Precision Scaling** - Re-scaling already scaled values leads to inflated amounts
5. **Mismatched Precision Scaling** - Different modules using different scaling methods (decimals vs hardcoded 1e18)
6. **Downcast Overflow** - Downcasting can silently overflow, breaking pre-downcast invariant checks
7. **Rounding Leaks Value From Protocol** - Fee calculations should round in favor of the protocol, not users
8. **Inverted Base/Rate Token Pairs** - Using opposite token pairs in calculations (e.g., WETH/DAI vs DAI/ETH)
9. **Decimal Assumption Errors** - Assuming all tokens have 18 decimals when some have 6, 8, or 2
10. **Interest Calculation Time Unit Confusion** - Mixing per-second and per-year rates without proper conversion
