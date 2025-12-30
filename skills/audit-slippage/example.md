# Code Examples: Slippage Protection Vulnerabilities

## Vulnerable Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB, uint liquidity);
}

interface IQuoter {
    function quoteExactInput(bytes memory path, uint256 amountIn) external returns (uint256 amountOut);
}

contract VulnerableSlippageExamples {
    IUniswapV2Router public router;
    IQuoter public quoter;
    address[] public path;

    // Pattern #1: No Slippage Parameter (Hard-coded Zero)
    function swap_NoSlippage_VULNERABLE(uint256 amountIn) external {
        router.swapExactTokensForTokens(
            amountIn,
            0, // No slippage protection - catastrophic!
            path,
            msg.sender,
            block.timestamp + 300
        );
    }

    // Pattern #2: No Expiration Deadline
    function swap_NoDeadline_VULNERABLE(uint256 amountIn, uint256 minOut) external {
        router.swapExactTokensForTokens(
            amountIn,
            minOut,
            path,
            msg.sender,
            type(uint256).max // Infinite deadline - no protection
        );
    }

    // Pattern #3: Block.timestamp as Deadline
    function swap_BlockTimestampDeadline_VULNERABLE(uint256 amountIn, uint256 minOut) external {
        router.swapExactTokensForTokens(
            amountIn,
            minOut,
            path,
            msg.sender,
            block.timestamp // Always valid - useless
        );
    }

    // Pattern #4: Incorrect Slippage Calculation (Wrong Reference)
    function swap_WrongSlippageBase_VULNERABLE(uint256 amountIn, uint256 slippageBps) external {
        // Calculating slippage from input instead of expected output
        uint256 minOut = amountIn * (10000 - slippageBps) / 10000; // Wrong!

        router.swapExactTokensForTokens(
            amountIn,
            minOut,
            path,
            msg.sender,
            block.timestamp + 300
        );
    }

    // Pattern #5: Mismatched Slippage Precision
    function swap_DecimalMismatch_VULNERABLE(uint256 amountInUSDC) external {
        // amountIn is 1000 USDC (6 decimals) = 1000e6
        // Expected output: 0.95 WETH (18 decimals) = 0.95e18
        uint256 minOut = amountInUSDC * 95 / 100; // Results in 950e6, not 0.95e18!

        router.swapExactTokensForTokens(
            amountInUSDC,
            minOut,
            path,
            msg.sender,
            block.timestamp + 300
        );
    }

    // Pattern #6: Hard-coded Slippage
    uint256 constant SLIPPAGE_BPS = 500; // 5% hard-coded

    function withdraw_HardcodedSlippage_VULNERABLE(uint256 expectedOut) external {
        uint256 minOut = expectedOut * (10000 - SLIPPAGE_BPS) / 10000;

        router.swapExactTokensForTokens(
            100 ether,
            minOut,
            path,
            msg.sender,
            block.timestamp + 300
        );
        // Fails if market moves >5%
    }

    // Pattern #7: MinTokensOut For Intermediate Amount
    function multiHopSwap_VULNERABLE(uint256 amountA, uint256 minB) external {
        // Swap A -> B
        address[] memory pathAB = new address[](2);
        pathAB[0] = address(0x1);
        pathAB[1] = address(0x2);

        uint256[] memory amountsB = router.swapExactTokensForTokens(
            amountA,
            minB, // Protected intermediate
            pathAB,
            address(this),
            block.timestamp + 300
        );

        // Swap B -> C
        address[] memory pathBC = new address[](2);
        pathBC[0] = address(0x2);
        pathBC[1] = address(0x3);

        router.swapExactTokensForTokens(
            amountsB[1],
            0, // No protection on final output!
            pathBC,
            msg.sender,
            block.timestamp + 300
        );
    }

    // Pattern #8: On-Chain Slippage Calculation
    function swap_OnChainQuoter_VULNERABLE(uint256 amountIn, bytes memory quotePath) external {
        // Quoter uses current reserves - manipulable via flash loan!
        uint256 expectedOut = quoter.quoteExactInput(quotePath, amountIn);
        uint256 minOut = expectedOut * 95 / 100;

        router.swapExactTokensForTokens(
            amountIn,
            minOut, // Based on manipulable price
            path,
            msg.sender,
            block.timestamp + 300
        );
    }

    // Pattern #9: Fixed Fee Tier Assumption
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }

    function swapV3_FixedFeeTier_VULNERABLE(uint256 amountIn, uint256 minOut) external pure returns (ExactInputSingleParams memory) {
        return ExactInputSingleParams({
            tokenIn: address(0x1),
            tokenOut: address(0x2),
            fee: 3000, // Assumes 0.3% pool - might have better liquidity at 0.05%
            recipient: address(this),
            deadline: block.timestamp + 300,
            amountIn: amountIn,
            amountOutMinimum: minOut,
            sqrtPriceLimitX96: 0
        });
    }

    // Pattern #11: No Slippage on Liquidity Operations
    function addLiquidity_NoSlippage_VULNERABLE(
        address tokenA,
        address tokenB,
        uint256 amountA,
        uint256 amountB
    ) external {
        router.addLiquidity(
            tokenA,
            tokenB,
            amountA,
            amountB,
            0, // amountAMin - no protection
            0, // amountBMin - no protection
            msg.sender,
            block.timestamp + 300
        );
    }

    // Pattern #13: Approval Race on Router Upgrade
    address public oldRouter;

    function upgradeRouter_ApprovalRace_VULNERABLE(address newRouter, address token) external {
        router = IUniswapV2Router(newRouter);
        // Old router still has approval - can be exploited!
        // Missing: approve(oldRouter, 0)
    }
}
```

## Fixed Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function decimals() external view returns (uint8);
}

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB, uint liquidity);
}

interface IUniswapV3Pool {
    function liquidity() external view returns (uint128);
}

interface IUniswapV3Factory {
    function getPool(address tokenA, address tokenB, uint24 fee) external view returns (address pool);
}

contract FixedSlippageExamples {
    IUniswapV2Router public router;
    IUniswapV3Factory public v3Factory;
    address[] public path;

    // Pattern #1 FIXED: User-Provided Slippage
    function swap_WithSlippage_FIXED(
        uint256 amountIn,
        uint256 minAmountOut,
        uint256 deadline
    ) external {
        require(minAmountOut > 0, "Invalid slippage");
        require(block.timestamp <= deadline, "Transaction expired");

        router.swapExactTokensForTokens(
            amountIn,
            minAmountOut, // User-provided protection
            path,
            msg.sender,
            deadline
        );
    }

    // Pattern #2 FIXED: User-Provided Deadline
    function swap_WithDeadline_FIXED(
        uint256 amountIn,
        uint256 minOut,
        uint256 deadline
    ) external {
        require(block.timestamp <= deadline, "Transaction expired");

        router.swapExactTokensForTokens(
            amountIn,
            minOut,
            path,
            msg.sender,
            deadline // User controls expiration
        );
    }

    // Pattern #3 FIXED: Accept Deadline Parameter
    function swap_ProperDeadline_FIXED(
        uint256 amountIn,
        uint256 minOut,
        uint256 deadline
    ) external {
        // Don't use block.timestamp - accept user deadline
        router.swapExactTokensForTokens(
            amountIn,
            minOut,
            path,
            msg.sender,
            deadline // From user, not block.timestamp
        );
    }

    // Pattern #4 FIXED: Calculate Slippage from Expected Output
    function swap_CorrectSlippageBase_FIXED(
        uint256 amountIn,
        uint256 expectedOut, // From off-chain quote
        uint256 slippageBps,
        uint256 deadline
    ) external {
        // Calculate from expected output, not input
        uint256 minOut = expectedOut * (10000 - slippageBps) / 10000;

        router.swapExactTokensForTokens(
            amountIn,
            minOut,
            path,
            msg.sender,
            deadline
        );
    }

    // Pattern #5 FIXED: Match Decimal Precision
    function swap_DecimalMatched_FIXED(
        uint256 amountInUSDC,
        uint256 expectedOutWETH, // Expected: 0.95e18
        uint256 slippageBps,
        uint256 deadline
    ) external {
        // expectedOutWETH is already in 18 decimals
        uint256 minOut = expectedOutWETH * (10000 - slippageBps) / 10000;
        // minOut preserves 18 decimals

        router.swapExactTokensForTokens(
            amountInUSDC,
            minOut,
            path,
            msg.sender,
            deadline
        );
    }

    // Pattern #6 FIXED: User-Controlled Slippage
    function withdraw_UserSlippage_FIXED(
        uint256 amount,
        uint256 expectedOut,
        uint256 slippageBps, // User specifies
        uint256 deadline
    ) external {
        uint256 minOut = expectedOut * (10000 - slippageBps) / 10000;

        router.swapExactTokensForTokens(
            amount,
            minOut,
            path,
            msg.sender,
            deadline
        );
    }

    // Pattern #7 FIXED: Protect Final Output in Multi-Hop
    function multiHopSwap_FIXED(
        uint256 amountA,
        uint256 minC, // Final output protection
        uint256 deadline
    ) external {
        // Use single multi-hop call with final slippage
        address[] memory pathAC = new address[](3);
        pathAC[0] = address(0x1); // A
        pathAC[1] = address(0x2); // B (intermediate)
        pathAC[2] = address(0x3); // C (final)

        router.swapExactTokensForTokens(
            amountA,
            minC, // Protects final output C
            pathAC,
            msg.sender,
            deadline
        );
    }

    // Pattern #8 FIXED: Off-Chain Slippage Calculation
    function swap_OffChainQuote_FIXED(
        uint256 amountIn,
        uint256 minAmountOut, // Calculated off-chain or via TWAP
        uint256 deadline
    ) external {
        // Accept pre-calculated minAmountOut from off-chain
        // or from TWAP oracle (not current block state)
        router.swapExactTokensForTokens(
            amountIn,
            minAmountOut,
            path,
            msg.sender,
            deadline
        );
    }

    // Pattern #9 FIXED: Query Multiple Fee Tiers
    function findBestFeeTier_FIXED(
        address tokenA,
        address tokenB
    ) public view returns (uint24 bestFee) {
        uint24[3] memory fees = [uint24(500), uint24(3000), uint24(10000)];
        uint128 maxLiquidity = 0;

        for (uint256 i = 0; i < fees.length; i++) {
            address pool = v3Factory.getPool(tokenA, tokenB, fees[i]);
            if (pool != address(0)) {
                uint128 liquidity = IUniswapV3Pool(pool).liquidity();
                if (liquidity > maxLiquidity) {
                    maxLiquidity = liquidity;
                    bestFee = fees[i];
                }
            }
        }

        require(maxLiquidity > 0, "No pool found");
        return bestFee;
    }

    // Pattern #11 FIXED: Slippage Protection on Liquidity Operations
    function addLiquidity_WithSlippage_FIXED(
        address tokenA,
        address tokenB,
        uint256 amountA,
        uint256 amountB,
        uint256 slippageBps,
        uint256 deadline
    ) external {
        uint256 amountAMin = amountA * (10000 - slippageBps) / 10000;
        uint256 amountBMin = amountB * (10000 - slippageBps) / 10000;

        router.addLiquidity(
            tokenA,
            tokenB,
            amountA,
            amountB,
            amountAMin, // Protected
            amountBMin, // Protected
            msg.sender,
            deadline
        );
    }

    // Pattern #13 FIXED: Revoke Old Approvals on Router Upgrade
    address public oldRouter;
    address public token;

    function upgradeRouter_Safe_FIXED(address newRouter) external {
        // Revoke old router approval first
        IERC20(token).approve(oldRouter, 0);

        // Update router
        oldRouter = address(router);
        router = IUniswapV2Router(newRouter);

        // Approve new router
        IERC20(token).approve(newRouter, type(uint256).max);
    }

    // Bonus: Emergency withdrawal with relaxed slippage
    bool public emergencyMode;

    function emergencyWithdraw_FIXED(
        uint256 amount,
        uint256 minOut,
        uint256 deadline
    ) external {
        require(emergencyMode, "Not emergency");
        // In emergency, use relaxed slippage (e.g., 20%)
        uint256 emergencyMinOut = minOut * 80 / 100;

        router.swapExactTokensForTokens(
            amount,
            emergencyMinOut,
            path,
            msg.sender,
            deadline
        );
    }

    // Bonus: Normalize decimals before slippage calculation
    function normalizeAmount_FIXED(
        address token,
        uint256 amount
    ) public view returns (uint256) {
        uint8 decimals = IERC20(token).decimals();
        if (decimals < 18) {
            return amount * (10 ** (18 - decimals));
        }
        return amount;
    }

    // Bonus: Helper for recommended slippage
    function getRecommendedSlippage_FIXED(
        address tokenA,
        address tokenB
    ) public pure returns (uint256 slippageBps) {
        // Stablecoin pairs: 0.5% (50 bps)
        // Volatile pairs: 2% (200 bps)
        // For demo - in production, query volatility metrics
        return 200; // 2% default for volatile pairs
    }
}
```
