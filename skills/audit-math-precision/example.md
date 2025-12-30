# Code Examples: Math Precision Vulnerabilities

## Vulnerable Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VulnerableMathExamples {
    uint256 constant PRECISION = 1e18;
    uint256 constant FEE_BPS = 500; // 5%

    // Pattern #1: Division Before Multiplication
    function calculateReward_VULNERABLE(uint256 amount, uint256 rate) public pure returns (uint256) {
        return (amount / PRECISION) * rate; // Loses precision
    }

    // Pattern #2: Rounding Down to Zero
    function collectFee_VULNERABLE(uint256 amount) public pure returns (uint256) {
        uint256 fee = (amount * FEE_BPS) / 10000;
        // If amount < 20, fee = 0 (free transactions)
        return fee;
    }

    // Pattern #3: No Precision Scaling (Mixed Decimals)
    function addLiquidity_VULNERABLE(uint256 amountUSDC, uint256 amountDAI) public pure returns (uint256) {
        // USDC is 6 decimals, DAI is 18 decimals
        return amountUSDC + amountDAI; // Magnitude error!
    }

    // Pattern #4: Excessive Precision Scaling
    function convertToken_VULNERABLE(uint256 amount) public pure returns (uint256) {
        // amount is already 1e18
        return amount * 1e18; // Inflates by 10^18!
    }

    // Pattern #5: Mismatched Precision Scaling
    function scaleAmount_VULNERABLE(uint256 amount) public pure returns (uint256) {
        // Assumes 18 decimals but token might be WBTC (8 decimals)
        return amount * 1e18;
    }

    // Pattern #6: Downcast Overflow
    struct Checkpoint {
        uint96 votes;
        uint32 blockNumber;
    }

    function setVotes_VULNERABLE(uint256 amount) public pure returns (uint96) {
        // Silent overflow if amount > type(uint96).max
        return uint96(amount);
    }

    // Pattern #7: Rounding Leaks Value (Protocol Fees)
    function calculateProtocolFee_VULNERABLE(uint256 amount, uint256 bps) public pure returns (uint256) {
        return (amount * bps) / 10000; // Rounds down, user pays less
    }

    // Pattern #8: Inverted Oracle Pairs
    function swapTokens_VULNERABLE(uint256 tokenAAmount, uint256 priceAinB) public pure returns (uint256) {
        // If we want B but oracle gives price of A in terms of B, this is wrong
        return tokenAAmount * priceAinB; // Should divide for inversion
    }

    // Pattern #9: Decimal Assumption Errors
    function calculateOneToken_VULNERABLE() public pure returns (uint256) {
        uint256 oneToken = 1 ether; // Assumes 18 decimals
        // Breaks if used with USDC (6) or WBTC (8)
        return oneToken;
    }

    // Pattern #10: Interest Calculation Time Unit Confusion
    function calculateInterest_VULNERABLE(
        uint256 principal,
        uint256 ratePerYear,
        uint256 lastUpdate
    ) public view returns (uint256) {
        uint256 timeElapsed = block.timestamp - lastUpdate;
        // ratePerYear is annual but timeElapsed is seconds - missing conversion!
        return principal * ratePerYear * timeElapsed;
    }

    // Pattern #11: Phantom Overflow (Unchecked Blocks)
    function addBalance_VULNERABLE(uint256 balance, uint256 amount) public pure returns (uint256) {
        unchecked {
            return balance + amount; // Can wrap to 0
        }
    }

    // Pattern #12: Loss of Precision in Exponentiation
    function compoundInterest_VULNERABLE(uint256 principal, uint256 rate, uint256 years)
        public pure returns (uint256)
    {
        return principal * (rate ** years); // Loses precision each iteration
    }

    // Pattern #13: Percentage Calculation Base Confusion
    function calculateFeeConfused_VULNERABLE(uint256 amount) public pure returns (uint256) {
        // Meant 5 bps (0.05%) but calculated 5%
        return (amount * 5) / 100;
    }
}
```

## Fixed Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

interface IERC20Extended {
    function decimals() external view returns (uint8);
}

contract FixedMathExamples {
    using SafeCast for uint256;

    uint256 constant PRECISION = 1e18;
    uint256 constant FEE_BPS = 500; // 5%
    uint256 constant SECONDS_PER_YEAR = 31536000;

    // Pattern #1 FIXED: Multiply First, Divide Last
    function calculateReward_FIXED(uint256 amount, uint256 rate) public pure returns (uint256) {
        return (amount * rate) / PRECISION; // Multiply first preserves precision
    }

    // Pattern #2 FIXED: Prevent Rounding to Zero
    function collectFee_FIXED(uint256 amount) public pure returns (uint256) {
        uint256 fee = (amount * FEE_BPS) / 10000;
        require(fee > 0, "Amount too small"); // Explicit minimum check
        return fee;
    }

    // Pattern #3 FIXED: Normalize Decimals Before Math
    function addLiquidity_FIXED(uint256 amountUSDC, uint256 amountDAI) public pure returns (uint256) {
        // Normalize USDC (6 decimals) to 18 decimals before adding
        uint256 normalizedUSDC = amountUSDC * 1e12; // 6 + 12 = 18
        return normalizedUSDC + amountDAI; // Both 18 decimals now
    }

    // Pattern #4 FIXED: Don't Double-Scale
    function convertToken_FIXED(uint256 amount) public pure returns (uint256) {
        // amount is already 1e18, use as-is
        return amount;
    }

    // Pattern #5 FIXED: Query Decimals Dynamically
    function scaleAmount_FIXED(address token, uint256 amount) public view returns (uint256) {
        uint8 decimals = IERC20Extended(token).decimals();
        if (decimals < 18) {
            return amount * (10 ** (18 - decimals));
        }
        return amount;
    }

    // Pattern #6 FIXED: Safe Downcast with Validation
    struct Checkpoint {
        uint96 votes;
        uint32 blockNumber;
    }

    function setVotes_FIXED(uint256 amount) public pure returns (uint96) {
        // Explicit bounds check before downcast
        require(amount <= type(uint96).max, "Amount exceeds uint96");
        return uint96(amount);

        // Or use OpenZeppelin SafeCast:
        // return amount.toUint96();
    }

    // Pattern #7 FIXED: Round Fees UP (Ceiling Math)
    function calculateProtocolFee_FIXED(uint256 amount, uint256 bps) public pure returns (uint256) {
        // Add (denominator - 1) before dividing to round up
        return (amount * bps + 9999) / 10000;
    }

    // Pattern #8 FIXED: Invert Oracle Price Correctly
    function swapTokens_FIXED(uint256 tokenAAmount, uint256 priceAinB) public pure returns (uint256) {
        // To get B from A when price is "A in terms of B", divide
        return (tokenAAmount * PRECISION) / priceAinB;
    }

    // Pattern #9 FIXED: Use Explicit Decimals Per Token
    uint256 constant USDC_DECIMALS = 6;
    uint256 constant WBTC_DECIMALS = 8;
    uint256 constant DAI_DECIMALS = 18;

    function calculateOneToken_FIXED(address token) public view returns (uint256) {
        uint8 decimals = IERC20Extended(token).decimals();
        return 10 ** decimals;
    }

    // Pattern #10 FIXED: Convert Time Units for Interest
    function calculateInterest_FIXED(
        uint256 principal,
        uint256 ratePerYear,
        uint256 lastUpdate
    ) public view returns (uint256) {
        uint256 timeElapsed = block.timestamp - lastUpdate;
        // Convert annual rate to per-second rate
        return (principal * ratePerYear * timeElapsed) / SECONDS_PER_YEAR;
    }

    // Pattern #11 FIXED: Only Use Unchecked When Provably Safe
    function addBalance_FIXED(uint256 balance, uint256 amount) public pure returns (uint256) {
        // Use checked arithmetic (default in 0.8+)
        return balance + amount; // Reverts on overflow

        // Only use unchecked for guaranteed-safe operations like:
        // unchecked { for (uint256 i = 0; i < array.length; ++i) { ... } }
    }

    // Pattern #12 FIXED: Iterative Multiplication with Scaling
    function compoundInterest_FIXED(uint256 principal, uint256 rate, uint256 years)
        public pure returns (uint256)
    {
        uint256 amount = principal;
        for (uint256 i = 0; i < years; i++) {
            amount = (amount * rate) / PRECISION;
        }
        return amount;
    }

    // Pattern #13 FIXED: Standardize on Basis Points
    function calculateFee_FIXED(uint256 amount) public pure returns (uint256) {
        // Use 10000 BPS consistently, document clearly
        uint256 FEE_5_BPS = 5; // 0.05% = 5 basis points
        return (amount * FEE_5_BPS) / 10000;
    }

    // Bonus: Helper function for ceiling division
    function ceilDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "Division by zero");
        return (a + b - 1) / b;
    }

    // Bonus: Safe multi-decimal normalization helper
    function normalizeDecimals(
        uint256 amount,
        uint8 fromDecimals,
        uint8 toDecimals
    ) internal pure returns (uint256) {
        if (fromDecimals == toDecimals) return amount;

        if (fromDecimals < toDecimals) {
            return amount * (10 ** (toDecimals - fromDecimals));
        } else {
            return amount / (10 ** (fromDecimals - toDecimals));
        }
    }
}
```
