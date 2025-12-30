# Auction Manipulation Vulnerability Examples

## Pattern #1: Self-Bidding to Reset Auction

### VULNERABLE
```solidity
contract VulnerableSelfBid {
    struct Auction {
        address borrower;
        uint256 startTime;
        uint256 duration;
        uint256 highestBid;
        address highestBidder;
    }

    mapping(uint256 => Auction) public auctions;

    // ISSUE: No check preventing borrower from bidding
    function bid(uint256 auctionId, uint256 amount) external {
        Auction storage auction = auctions[auctionId];
        require(block.timestamp < auction.startTime + auction.duration, "Ended");
        require(amount > auction.highestBid, "Bid too low");

        auction.highestBid = amount;
        auction.highestBidder = msg.sender;

        // Borrower can bid on own loan
        // Each bid extends "effective" auction by resetting competitive pressure
    }

    function settleAuction(uint256 auctionId) external {
        Auction storage auction = auctions[auctionId];
        require(block.timestamp >= auction.startTime + auction.duration, "Active");

        // Transfer to highest bidder
        // If borrower is highest bidder, they "buy" own loan
        // Can repeat by refinancing and starting new auction
    }

    // Attack:
    // 1. Auction starts for underwater loan
    // 2. Legitimate bidder bids 80 ETH
    // 3. Borrower bids 81 ETH (buying own loan)
    // 4. Auction ends, borrower "pays" themselves
    // 5. Borrower can refinance and repeat
    // 6. Extends underwater position indefinitely
}
```

### FIXED
```solidity
contract FixedSelfBid {
    struct Auction {
        address borrower;
        uint256 startTime;
        uint256 duration;
        uint256 highestBid;
        address highestBidder;
    }

    mapping(uint256 => Auction) public auctions;

    function bid(uint256 auctionId, uint256 amount) external {
        Auction storage auction = auctions[auctionId];
        require(block.timestamp < auction.startTime + auction.duration, "Ended");
        require(amount > auction.highestBid, "Bid too low");

        // Prevent self-bidding
        require(msg.sender != auction.borrower, "Cannot bid on own auction");

        auction.highestBid = amount;
        auction.highestBidder = msg.sender;
    }
}
```

## Pattern #2: Auction Start During Sequencer Downtime

### VULNERABLE
```solidity
contract VulnerableL2Auction {
    struct Auction {
        uint256 startTime;
        uint256 duration;
    }

    mapping(uint256 => Auction) public auctions;

    // ISSUE: On Arbitrum/Optimism, no sequencer check
    function startAuction(uint256 loanId) external {
        auctions[loanId] = Auction({
            startTime: block.timestamp,
            duration: 24 hours
        });

        // Scenario:
        // 1. Sequencer down for 2 hours
        // 2. Auction starts at T=0 (in transaction queue)
        // 3. Sequencer restarts at T=2h
        // 4. Auction processes with startTime = T=0
        // 5. First bidder after restart has only 22h window
        // 6. Unfair advantage to whoever can bid first after restart
    }
}
```

### FIXED
```solidity
contract FixedL2Auction {
    AggregatorV3Interface public sequencerUptimeFeed;
    uint256 public constant GRACE_PERIOD = 1 hours;

    struct Auction {
        uint256 startTime;
        uint256 duration;
    }

    mapping(uint256 => Auction) public auctions;

    function startAuction(uint256 loanId) external {
        // Check sequencer status on L2
        (, int256 answer, uint256 startedAt, , ) = sequencerUptimeFeed.latestRoundData();

        // Sequencer must be up (answer == 0)
        require(answer == 0, "Sequencer down");

        // Grace period after sequencer restart
        require(
            block.timestamp >= startedAt + GRACE_PERIOD,
            "Grace period active"
        );

        // Now safe to start auction
        auctions[loanId] = Auction({
            startTime: block.timestamp,
            duration: 24 hours
        });
    }
}
```

## Pattern #3: Insufficient Auction Length Validation

### VULNERABLE
```solidity
contract VulnerableLength {
    struct Auction {
        uint256 startTime;
        uint256 duration; // No minimum!
    }

    mapping(uint256 => Auction) public auctions;

    // ISSUE: No minimum auction length
    function startAuction(uint256 loanId, uint256 duration) external {
        auctions[loanId] = Auction({
            startTime: block.timestamp,
            duration: duration // Can be 1 second!
        });
    }

    function seizeCollateral(uint256 loanId) external {
        Auction storage auction = auctions[loanId];
        require(block.timestamp >= auction.startTime + auction.duration, "Active");

        // Transfer collateral to caller
    }

    // Attack:
    // 1. Liquidator starts auction with duration = 1 second
    // 2. Waits 1 second
    // 3. Immediately seizes collateral
    // 4. No competitive bidding occurred
    // 5. Borrower gets unfair price
}
```

### FIXED
```solidity
contract FixedLength {
    uint256 public constant MIN_AUCTION_DURATION = 1 hours;
    uint256 public constant MAX_AUCTION_DURATION = 7 days;

    struct Auction {
        uint256 startTime;
        uint256 duration;
    }

    mapping(uint256 => Auction) public auctions;

    function startAuction(uint256 loanId, uint256 duration) external {
        // Enforce minimum and maximum
        require(
            duration >= MIN_AUCTION_DURATION,
            "Duration too short"
        );
        require(
            duration <= MAX_AUCTION_DURATION,
            "Duration too long"
        );

        auctions[loanId] = Auction({
            startTime: block.timestamp,
            duration: duration
        });
    }

    // Minimum 1 hour ensures competitive bidding
}
```

## Pattern #4: Auction Can Be Seized During Active Period

### VULNERABLE
```solidity
contract VulnerableOffByOne {
    struct Auction {
        uint256 startTime;
        uint256 duration;
    }

    mapping(uint256 => Auction) public auctions;

    function seizeCollateral(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // ISSUE: Using > instead of >=
        require(
            block.timestamp > auction.startTime + auction.duration,
            "Auction active"
        );

        // Seizure allowed
    }

    // Problem:
    // If auction ends at timestamp T
    // This check allows seizure at timestamp T
    // But auction should be active until T inclusive
    // Bidder at exactly T gets front-run by seize

    // Example:
    // startTime = 1000
    // duration = 3600 (1 hour)
    // endTime = 4600
    //
    // At timestamp 4600:
    // - Bidder submits bid (expects auction active)
    // - Seize transaction front-runs with timestamp 4600
    // - 4600 > 4600 is FALSE
    // - Wait, 4600 > 4600 is FALSE but should be checked at 4601
    //
    // Actually at timestamp 4601:
    // - 4601 > 4600 is TRUE
    // - Auction can be seized
    // - But bidder at 4600 thought they had until 4600 inclusive
}
```

### FIXED
```solidity
contract FixedOffByOne {
    struct Auction {
        uint256 startTime;
        uint256 duration;
    }

    mapping(uint256 => Auction) public auctions;

    function seizeCollateral(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // Use >= to ensure auction fully complete
        require(
            block.timestamp >= auction.startTime + auction.duration,
            "Auction active"
        );

        // Now seizure only allowed after auction end
    }

    // At endTime = 4600:
    // - 4600 >= 4600 is TRUE
    // - But bids should still be accepted at 4600
    //
    // Better: bid check uses <, seize uses >=
    function bid(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // Bid allowed before end
        require(
            block.timestamp < auction.startTime + auction.duration,
            "Auction ended"
        );
    }

    // Timeline:
    // T < 4600: Bids allowed, seize not allowed
    // T = 4600: Bids not allowed, seize not allowed (grace)
    // T >= 4600: Bids not allowed, seize allowed
    //
    // Actually at T=4600:
    // bid: 4600 < 4600 = FALSE (no bid)
    // seize: 4600 >= 4600 = TRUE (can seize)
    //
    // Issue: No buffer at exact boundary
    // Solution: Add buffer period
}
```

### BEST PRACTICE
```solidity
contract BestPracticeTimestamp {
    struct Auction {
        uint256 startTime;
        uint256 duration;
    }

    mapping(uint256 => Auction) public auctions;

    function bid(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // Strict: bid allowed only before end
        require(
            block.timestamp < auction.startTime + auction.duration,
            "Auction ended"
        );
    }

    function seizeCollateral(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // Strict: seize allowed only after end
        require(
            block.timestamp > auction.startTime + auction.duration,
            "Auction active"
        );

        // This creates 1-second gap at exact boundary
        // At T = endTime:
        //   - bid: T < endTime = FALSE (rejected)
        //   - seize: T > endTime = FALSE (rejected)
        // At T = endTime + 1:
        //   - seize: T > endTime = TRUE (allowed)
    }
}
```

## Complete Auction Example

### VULNERABLE
```solidity
contract CompleteVulnerableAuction {
    struct Auction {
        address borrower;
        uint256 startTime;
        uint256 duration; // No minimum
        uint256 highestBid;
        address highestBidder;
    }

    mapping(uint256 => Auction) public auctions;

    function startAuction(uint256 loanId, uint256 duration) external {
        // No sequencer check on L2
        // No minimum duration
        auctions[loanId] = Auction({
            borrower: msg.sender,
            startTime: block.timestamp,
            duration: duration,
            highestBid: 0,
            highestBidder: address(0)
        });
    }

    function bid(uint256 loanId, uint256 amount) external {
        Auction storage auction = auctions[loanId];

        // No self-bidding prevention
        require(block.timestamp < auction.startTime + auction.duration, "Ended");

        auction.highestBid = amount;
        auction.highestBidder = msg.sender;
    }

    function settle(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // Off-by-one: uses > not >=
        require(block.timestamp > auction.startTime + auction.duration, "Active");

        // Transfer to highest bidder
    }
}
```

### FIXED
```solidity
contract CompleteFixedAuction {
    AggregatorV3Interface public sequencerUptimeFeed;
    uint256 public constant MIN_DURATION = 1 hours;
    uint256 public constant MAX_DURATION = 7 days;
    uint256 public constant GRACE_PERIOD = 1 hours;

    struct Auction {
        address borrower;
        uint256 startTime;
        uint256 duration;
        uint256 highestBid;
        address highestBidder;
    }

    mapping(uint256 => Auction) public auctions;

    function startAuction(uint256 loanId, uint256 duration) external {
        // L2 sequencer check
        if (address(sequencerUptimeFeed) != address(0)) {
            (, int256 answer, uint256 startedAt, , ) = sequencerUptimeFeed.latestRoundData();
            require(answer == 0, "Sequencer down");
            require(
                block.timestamp >= startedAt + GRACE_PERIOD,
                "Grace period"
            );
        }

        // Duration bounds
        require(duration >= MIN_DURATION, "Too short");
        require(duration <= MAX_DURATION, "Too long");

        auctions[loanId] = Auction({
            borrower: msg.sender,
            startTime: block.timestamp,
            duration: duration,
            highestBid: 0,
            highestBidder: address(0)
        });
    }

    function bid(uint256 loanId, uint256 amount) external {
        Auction storage auction = auctions[loanId];

        // Prevent self-bidding
        require(msg.sender != auction.borrower, "No self-bid");

        // Proper timestamp check
        require(block.timestamp < auction.startTime + auction.duration, "Ended");
        require(amount > auction.highestBid, "Bid too low");

        auction.highestBid = amount;
        auction.highestBidder = msg.sender;
    }

    function settle(uint256 loanId) external {
        Auction storage auction = auctions[loanId];

        // Correct comparison (no off-by-one)
        require(
            block.timestamp > auction.startTime + auction.duration,
            "Active"
        );

        // Transfer to highest bidder
    }
}
```

## Summary: Key Protections

1. **Self-bidding prevention:** Borrower cannot bid on own auction
2. **Sequencer checks:** Validate uptime + grace period on L2
3. **Duration bounds:** Minimum 1 hour, maximum 7 days
4. **Correct comparisons:** bid uses <, settle uses > (no overlap)

## Timestamp Comparison Reference

```solidity
// Correct pattern:
function canBid() public view returns (bool) {
    return block.timestamp < endTime; // Strict before
}

function canSettle() public view returns (bool) {
    return block.timestamp > endTime; // Strict after
}

// At T = endTime:
// canBid() = false
// canSettle() = false
// (1-second gap is acceptable)

// At T = endTime + 1:
// canBid() = false
// canSettle() = true
```
