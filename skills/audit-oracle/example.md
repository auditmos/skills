# Oracle Integration Vulnerability Examples

## Pattern #1: Not Checking Stale Prices

### VULNERABLE
```solidity
contract VulnerableStalePrice {
    AggregatorV3Interface internal priceFeed;

    function getPrice() public view returns (uint256) {
        // ISSUE: No staleness check on updatedAt
        (, int256 price, , , ) = priceFeed.latestRoundData();

        // During oracle outage or high volatility:
        // - updatedAt could be hours old
        // - price no longer reflects market
        // - enables arbitrage and unfair liquidations

        return uint256(price);
    }
}
```

### FIXED
```solidity
contract FixedStalePrice {
    AggregatorV3Interface internal priceFeed;
    uint256 public constant HEARTBEAT = 3600; // 1 hour for ETH/USD

    function getPrice() public view returns (uint256) {
        (, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();

        // Verify price is fresh
        require(
            block.timestamp - updatedAt <= HEARTBEAT,
            "Stale price"
        );

        return uint256(price);
    }
}
```

## Pattern #2: Missing L2 Sequencer Check

### VULNERABLE
```solidity
contract VulnerableL2Oracle {
    AggregatorV3Interface internal priceFeed;

    // ISSUE: On Arbitrum/Optimism, no sequencer check
    function getCollateralValue(address user) public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();

        // During sequencer downtime:
        // - Oracle updates stop
        // - Prices become stale
        // - Users liquidated unfairly when sequencer restarts

        return userCollateral[user] * uint256(price) / 1e8;
    }
}
```

### FIXED
```solidity
contract FixedL2Oracle {
    AggregatorV3Interface internal priceFeed;
    AggregatorV3Interface internal sequencerUptimeFeed;
    uint256 public constant GRACE_PERIOD = 3600; // 1 hour

    function getCollateralValue(address user) public view returns (uint256) {
        // Check sequencer status
        (, int256 answer, uint256 startedAt, , ) = sequencerUptimeFeed.latestRoundData();

        // 0 = sequencer up, 1 = sequencer down
        require(answer == 0, "Sequencer down");

        // Grace period after sequencer restart
        require(
            block.timestamp - startedAt > GRACE_PERIOD,
            "Grace period active"
        );

        // Now safe to use price
        (, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();
        require(block.timestamp - updatedAt <= 3600, "Stale price");

        return userCollateral[user] * uint256(price) / 1e8;
    }
}
```

## Pattern #3: Same Heartbeat For Multiple Feeds

### VULNERABLE
```solidity
contract VulnerableSameHeartbeat {
    AggregatorV3Interface internal btcFeed;
    AggregatorV3Interface internal usdcFeed;
    uint256 public constant HEARTBEAT = 3600; // 1 hour - WRONG for USDC

    function getPrices() public view returns (uint256 btc, uint256 usdc) {
        // ISSUE: BTC feed has 1h heartbeat, but USDC has 24h heartbeat
        (, int256 btcPrice, , uint256 btcUpdated, ) = btcFeed.latestRoundData();
        (, int256 usdcPrice, , uint256 usdcUpdated, ) = usdcFeed.latestRoundData();

        // USDC check will fail incorrectly - 24h is normal for stablecoins
        require(block.timestamp - btcUpdated <= HEARTBEAT, "BTC stale");
        require(block.timestamp - usdcUpdated <= HEARTBEAT, "USDC stale"); // FALSE POSITIVE

        return (uint256(btcPrice), uint256(usdcPrice));
    }
}
```

### FIXED
```solidity
contract FixedPerFeedHeartbeat {
    AggregatorV3Interface internal btcFeed;
    AggregatorV3Interface internal usdcFeed;

    // Feed-specific heartbeats from Chainlink docs
    uint256 public constant BTC_HEARTBEAT = 3600; // 1 hour
    uint256 public constant USDC_HEARTBEAT = 86400; // 24 hours

    function getPrices() public view returns (uint256 btc, uint256 usdc) {
        (, int256 btcPrice, , uint256 btcUpdated, ) = btcFeed.latestRoundData();
        (, int256 usdcPrice, , uint256 usdcUpdated, ) = usdcFeed.latestRoundData();

        require(block.timestamp - btcUpdated <= BTC_HEARTBEAT, "BTC stale");
        require(block.timestamp - usdcUpdated <= USDC_HEARTBEAT, "USDC stale");

        return (uint256(btcPrice), uint256(usdcPrice));
    }
}
```

## Pattern #4: Assuming Oracle Precision

### VULNERABLE
```solidity
contract VulnerableAssumedDecimals {
    AggregatorV3Interface internal priceFeed;

    function getCollateralValue(uint256 collateralAmount) public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();

        // ISSUE: Assumes 18 decimals but Chainlink feeds return 8
        // collateralAmount is in 18 decimals
        // price is in 8 decimals
        // Result off by 10^10

        return collateralAmount * uint256(price); // WRONG
    }
}
```

### FIXED
```solidity
contract FixedOracleDecimals {
    AggregatorV3Interface internal priceFeed;

    function getCollateralValue(uint256 collateralAmount) public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        uint8 decimals = priceFeed.decimals(); // Usually 8

        // Scale price to 18 decimals to match collateralAmount
        uint256 scaledPrice = uint256(price) * 10 ** (18 - decimals);

        return collateralAmount * scaledPrice / 1e18;
    }
}
```

## Pattern #6: Unhandled Oracle Reverts

### VULNERABLE
```solidity
contract VulnerableOracleRevert {
    AggregatorV3Interface internal priceFeed;

    function liquidate(address user) external {
        // ISSUE: If oracle reverts, entire protocol DoS
        (, int256 price, , , ) = priceFeed.latestRoundData();

        uint256 collateralValue = getCollateral(user) * uint256(price) / 1e8;

        // Oracle maintenance/failure = complete protocol freeze
        // Nobody can liquidate, withdraw, or perform any action
    }
}
```

### FIXED
```solidity
contract FixedOracleRevert {
    AggregatorV3Interface internal priceFeed;
    AggregatorV3Interface internal backupFeed;
    uint256 public lastKnownPrice;

    function getPrice() public returns (uint256) {
        try priceFeed.latestRoundData() returns (
            uint80,
            int256 price,
            uint256,
            uint256 updatedAt,
            uint80
        ) {
            require(block.timestamp - updatedAt <= 3600, "Stale");
            lastKnownPrice = uint256(price);
            return uint256(price);
        } catch {
            // Try backup oracle
            try backupFeed.latestRoundData() returns (
                uint80,
                int256 backupPrice,
                uint256,
                uint256 backupUpdated,
                uint80
            ) {
                require(block.timestamp - backupUpdated <= 3600, "Backup stale");
                lastKnownPrice = uint256(backupPrice);
                return uint256(backupPrice);
            } catch {
                // Use last known price with warning
                require(lastKnownPrice > 0, "No price available");
                return lastKnownPrice;
            }
        }
    }
}
```

## Pattern #7: Unhandled Depeg Events

### VULNERABLE
```solidity
contract VulnerableDepeg {
    AggregatorV3Interface internal btcFeed; // BTC/USD feed

    function getWBTCValue(uint256 wbtcAmount) public view returns (uint256) {
        // ISSUE: Uses BTC/USD for WBTC without checking peg
        (, int256 btcPrice, , , ) = btcFeed.latestRoundData();

        // If WBTC bridge compromised and depegs:
        // - WBTC trades at $10k but BTC at $60k
        // - Oracle shows $60k for worthless WBTC
        // - Protocol becomes insolvent

        return wbtcAmount * uint256(btcPrice) / 1e8;
    }
}
```

### FIXED
```solidity
contract FixedDepeg {
    AggregatorV3Interface internal btcFeed; // BTC/USD
    AggregatorV3Interface internal wbtcBtcFeed; // WBTC/BTC peg feed
    uint256 public constant MIN_PEG = 0.98e8; // 98% - 2% depeg tolerance

    function getWBTCValue(uint256 wbtcAmount) public view returns (uint256) {
        // Get BTC price
        (, int256 btcPrice, , uint256 btcUpdated, ) = btcFeed.latestRoundData();
        require(block.timestamp - btcUpdated <= 3600, "BTC price stale");

        // Check WBTC peg to BTC
        (, int256 pegPrice, , uint256 pegUpdated, ) = wbtcBtcFeed.latestRoundData();
        require(block.timestamp - pegUpdated <= 3600, "Peg price stale");

        // Verify peg is maintained
        require(pegPrice >= int256(MIN_PEG), "WBTC depegged");

        // Use lower of: BTC price or WBTC/BTC ratio * BTC price
        uint256 wbtcPrice = uint256(btcPrice) * uint256(pegPrice) / 1e8;

        return wbtcAmount * wbtcPrice / 1e8;
    }
}
```

## Pattern #9: Using Slot0 Price

### VULNERABLE
```solidity
contract VulnerableSlot0 {
    IUniswapV3Pool public pool;

    function getPrice() public view returns (uint256) {
        // ISSUE: slot0 is spot price, manipulable in single tx
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();

        // Flash loan attack:
        // 1. Borrow large amount
        // 2. Swap to manipulate pool price
        // 3. Call getPrice() - returns manipulated value
        // 4. Exploit overvalued collateral
        // 5. Repay flash loan
        // All in one transaction!

        uint256 price = (uint256(sqrtPriceX96) * uint256(sqrtPriceX96) * 1e18) >> 192;
        return price;
    }
}
```

### FIXED
```solidity
contract FixedTWAP {
    IUniswapV3Pool public pool;
    uint32 public constant TWAP_INTERVAL = 1800; // 30 minutes

    function getPrice() public view returns (uint256) {
        // Use TWAP instead of spot price
        uint32[] memory secondsAgos = new uint32[](2);
        secondsAgos[0] = TWAP_INTERVAL;
        secondsAgos[1] = 0;

        (int56[] memory tickCumulatives, ) = pool.observe(secondsAgos);

        // Calculate time-weighted average tick
        int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
        int24 arithmeticMeanTick = int24(tickCumulativesDelta / int56(uint56(TWAP_INTERVAL)));

        // Convert tick to price
        uint160 sqrtPriceX96 = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);
        uint256 price = (uint256(sqrtPriceX96) * uint256(sqrtPriceX96) * 1e18) >> 192;

        // TWAP cannot be manipulated in single transaction
        return price;
    }
}
```

## Pattern #10: Price Feed Direction Confusion

### VULNERABLE
```solidity
contract VulnerablePriceDirection {
    AggregatorV3Interface internal daiFeed; // DAI/USD feed

    function getDaiPerEth() public view returns (uint256) {
        // ISSUE: Need ETH/DAI but using DAI/USD
        (, int256 daiPrice, , , ) = daiFeed.latestRoundData(); // $1.00

        // daiPrice = 1.00 (DAI per USD)
        // Need: DAI per ETH
        // Should be: ETH/USD ÷ DAI/USD = 3000 / 1 = 3000 DAI per ETH
        // Code returns: 1 DAI per ETH - WRONG by 3000x

        return uint256(daiPrice);
    }
}
```

### FIXED
```solidity
contract FixedPriceDirection {
    AggregatorV3Interface internal ethFeed; // ETH/USD
    AggregatorV3Interface internal daiFeed; // DAI/USD

    function getDaiPerEth() public view returns (uint256) {
        (, int256 ethPrice, , , ) = ethFeed.latestRoundData(); // $3000
        (, int256 daiPrice, , , ) = daiFeed.latestRoundData(); // $1.00

        uint8 ethDecimals = ethFeed.decimals(); // 8
        uint8 daiDecimals = daiFeed.decimals(); // 8

        // DAI per ETH = (ETH/USD) / (DAI/USD)
        // = 3000 / 1 = 3000 DAI per ETH
        uint256 daiPerEth = (uint256(ethPrice) * 10 ** daiDecimals) / uint256(daiPrice);

        return daiPerEth;
    }
}
```

## Pattern #11: Missing Circuit Breaker Checks

### VULNERABLE
```solidity
contract VulnerableCircuitBreaker {
    AggregatorV3Interface internal priceFeed;

    function getPrice() public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();

        // ISSUE: During flash crash, Chainlink returns minAnswer/maxAnswer
        // E.g., ETH flash crashes to $1, but oracle has minAnswer = $100
        // Oracle returns $100 (circuit breaker), not $1
        // Protocol treats $100 as real price - incorrect collateral valuation

        return uint256(price);
    }
}
```

### FIXED
```solidity
contract FixedCircuitBreaker {
    AggregatorV3Interface internal priceFeed;

    function getPrice() public view returns (uint256) {
        AggregatorV2V3Interface aggregator = AggregatorV2V3Interface(address(priceFeed));

        (, int256 price, , , ) = priceFeed.latestRoundData();

        // Get circuit breaker bounds
        int192 minAnswer = aggregator.minAnswer();
        int192 maxAnswer = aggregator.maxAnswer();

        // Verify price not at bounds
        require(
            price > minAnswer && price < maxAnswer,
            "Circuit breaker triggered"
        );

        // During extreme events, revert rather than use bound value
        // Forces manual intervention or fallback pricing

        return uint256(price);
    }
}
```

## Summary: Key Protections

1. **Staleness checks:** updatedAt vs feed-specific heartbeat
2. **L2 sequencer:** Check uptime + grace period on Arbitrum/Optimism
3. **Decimals:** Call decimals(), never assume 8 or 18
4. **Error handling:** try/catch with fallback oracle
5. **Depeg monitoring:** Separate feeds for wrapped assets
6. **TWAP:** Use time-weighted average, not slot0 spot price
7. **Direction:** Verify quote/base order, calculate correctly
8. **Circuit breakers:** Check price != minAnswer/maxAnswer

## Feed-Specific Heartbeats (Ethereum Mainnet)

- ETH/USD: 3600s (1 hour)
- BTC/USD: 3600s (1 hour)
- USDC/USD: 86400s (24 hours)
- USDT/USD: 86400s (24 hours)
- DAI/USD: 3600s (1 hour)
- LINK/USD: 3600s (1 hour)

Always verify current values in Chainlink docs.
