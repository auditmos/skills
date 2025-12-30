# Lending & Borrowing Vulnerability Examples

## Pattern #1: Liquidation Before Default

### VULNERABLE
```solidity
contract VulnerableLending {
    struct Loan {
        uint256 startTime;
        uint256 paymentCycleDuration; // e.g., 30 days
        uint256 paymentDefaultDuration; // e.g., 7 days
    }

    // ISSUE: Can liquidate during payment cycle if paymentDefaultDuration < paymentCycleDuration
    function liquidate(uint256 loanId) external {
        Loan memory loan = loans[loanId];
        require(
            block.timestamp > loan.startTime + loan.paymentDefaultDuration,
            "Not defaulted"
        );
        // Liquidate even though payment not due yet
    }
}
```

### FIXED
```solidity
contract FixedLending {
    struct Loan {
        uint256 lastPaymentTime;
        uint256 paymentCycleDuration;
        uint256 gracePeriod;
    }

    function liquidate(uint256 loanId) external {
        Loan memory loan = loans[loanId];
        uint256 paymentDueDate = loan.lastPaymentTime + loan.paymentCycleDuration;
        require(
            block.timestamp > paymentDueDate + loan.gracePeriod,
            "Not defaulted - payment not due or grace period active"
        );
        // Only liquidate after payment due + grace period
    }
}
```

## Pattern #2: Collateral Manipulation Preventing Liquidation

### VULNERABLE
```solidity
contract VulnerableCollateral {
    mapping(address => uint256) public collateral;

    // ISSUE: Borrower can zero out collateral record
    function updateCollateral(uint256 amount) external {
        collateral[msg.sender] = amount; // Can set to 0!
    }

    function liquidate(address borrower) external {
        require(collateral[borrower] > 0, "No collateral");
        // Never executes if borrower set collateral to 0
    }
}
```

### FIXED
```solidity
contract FixedCollateral {
    mapping(address => uint256) public collateral;
    mapping(address => bool) public hasActiveLoan;

    function updateCollateral(uint256 amount) external {
        require(!hasActiveLoan[msg.sender], "Cannot modify with active loan");
        require(amount >= collateral[msg.sender], "Can only increase");
        collateral[msg.sender] = amount;
    }

    function liquidate(address borrower) external {
        require(hasActiveLoan[borrower], "No active loan");
        require(collateral[borrower] > 0, "No collateral");
        // Protected: collateral cannot be zeroed with active loan
    }
}
```

## Pattern #3: Loan Closure Without Repayment

### VULNERABLE
```solidity
contract VulnerableLoanClosure {
    uint256 public outstandingLoans;
    mapping(uint256 => uint256) public loanDebt;

    // ISSUE: Doesn't validate loan exists or is repaid
    function close(uint256 loanId) external {
        outstandingLoans--; // Decrements even for non-existent ID
        delete loanDebt[loanId];
    }
}
```

### FIXED
```solidity
contract FixedLoanClosure {
    uint256 public outstandingLoans;
    mapping(uint256 => uint256) public loanDebt;
    mapping(uint256 => bool) public loanExists;

    function close(uint256 loanId) external {
        require(loanExists[loanId], "Loan does not exist");
        require(loanDebt[loanId] == 0, "Debt not fully repaid");

        outstandingLoans--;
        loanExists[loanId] = false;
        delete loanDebt[loanId];
    }
}
```

## Pattern #4: Asymmetric Pause Mechanism

### VULNERABLE
```solidity
contract VulnerableAsymmetricPause {
    bool public repaymentsPaused;

    function repay(uint256 loanId) external {
        require(!repaymentsPaused, "Repayments paused");
        // Process repayment
    }

    // ISSUE: Liquidations work even when repayments paused
    function liquidate(uint256 loanId) external {
        // No pause check - borrowers can't repay but can be liquidated!
    }
}
```

### FIXED
```solidity
contract FixedSymmetricPause {
    bool public operationsPaused;

    function repay(uint256 loanId) external {
        require(!operationsPaused, "Operations paused");
        // Process repayment
    }

    function liquidate(uint256 loanId) external {
        require(!operationsPaused, "Operations paused");
        // Symmetric: both paused together
    }
}
```

## Pattern #5: Token Disallow Blocks Existing Operations

### VULNERABLE
```solidity
contract VulnerableTokenDisallow {
    mapping(address => bool) public allowedTokens;

    function setTokenAllowed(address token, bool allowed) external onlyOwner {
        allowedTokens[token] = allowed;
    }

    // ISSUE: Disallowing token prevents existing loans from being repaid
    function repay(uint256 loanId) external {
        Loan memory loan = loans[loanId];
        require(allowedTokens[loan.token], "Token not allowed");
        // Existing loans become unrepayable if token disallowed!
    }
}
```

### FIXED
```solidity
contract FixedTokenDisallow {
    mapping(address => bool) public allowedTokens;
    mapping(uint256 => bool) public loanExists;

    function setTokenAllowed(address token, bool allowed) external onlyOwner {
        allowedTokens[token] = allowed;
        // Only affects NEW loans
    }

    function repay(uint256 loanId) external {
        require(loanExists[loanId], "Loan does not exist");
        // No token allowlist check - existing loans always repayable
        Loan memory loan = loans[loanId];
        // Process repayment
    }

    function createLoan(address token, uint256 amount) external {
        require(allowedTokens[token], "Token not allowed");
        // Allowlist only checked at creation
    }
}
```

## Pattern #6: No Grace Period After Unpause

### VULNERABLE
```solidity
contract VulnerableGracePeriod {
    bool public paused;
    uint256 public lastUnpauseTime;

    function liquidate(uint256 loanId) external {
        require(!paused, "Paused");
        // ISSUE: Immediately liquidates after unpause
        Loan memory loan = loans[loanId];
        require(block.timestamp > loan.dueDate, "Not due");
        // Liquidates borrowers who couldn't repay during pause
    }
}
```

### FIXED
```solidity
contract FixedGracePeriod {
    bool public paused;
    uint256 public lastUnpauseTime;
    uint256 public constant UNPAUSE_GRACE_PERIOD = 1 days;

    function liquidate(uint256 loanId) external {
        require(!paused, "Paused");
        require(
            block.timestamp > lastUnpauseTime + UNPAUSE_GRACE_PERIOD,
            "Grace period active"
        );

        Loan memory loan = loans[loanId];
        require(block.timestamp > loan.dueDate, "Not due");
        // Grace period gives borrowers time to repay after unpause
    }
}
```

## Pattern #7: Incorrect Liquidation Share Calculations

### VULNERABLE
```solidity
contract VulnerableLiquidationShares {
    function liquidate(uint256 loanId, uint256 repayAmount) external {
        Loan memory loan = loans[loanId];

        // ISSUE: Share calculated from single loan debt, not total position
        uint256 collateralShare = (repayAmount * loan.collateral) / loan.debt;

        // Attacker repays tiny amount to drain all collateral
        transferCollateral(msg.sender, collateralShare);
    }
}
```

### FIXED
```solidity
contract FixedLiquidationShares {
    function liquidate(uint256 loanId, uint256 repayAmount) external {
        Loan memory loan = loans[loanId];
        uint256 totalDebt = getTotalDebt(loan.borrower); // All loans

        // Share calculated from total outstanding debt
        uint256 collateralShare = (repayAmount * loan.collateral) / totalDebt;

        require(repayAmount >= totalDebt, "Must repay full debt for collateral");
        transferCollateral(msg.sender, collateralShare);
    }
}
```

## Pattern #8: Repayments Sent to Zero Address

### VULNERABLE
```solidity
contract VulnerableRepaymentRouting {
    mapping(uint256 => address) public loanLender;

    function repay(uint256 loanId, uint256 amount) external {
        address lender = loanLender[loanId]; // Could be address(0) if deleted
        // ISSUE: No validation - sends to zero address
        token.transfer(lender, amount);
    }
}
```

### FIXED
```solidity
contract FixedRepaymentRouting {
    mapping(uint256 => address) public loanLender;
    mapping(uint256 => bool) public loanActive;

    function repay(uint256 loanId, uint256 amount) external {
        require(loanActive[loanId], "Loan not active");
        address lender = loanLender[loanId];
        require(lender != address(0), "Invalid lender");

        token.transfer(lender, amount);
    }
}
```

## Pattern #9: Forced Loan Assignment

### VULNERABLE
```solidity
contract VulnerableForcedLoan {
    mapping(uint256 => address) public loanLender;

    // ISSUE: Anyone can force loans onto unwilling lenders
    function buyLoan(uint256 loanId, address newLender) external {
        loanLender[loanId] = newLender; // No consent required!
    }
}
```

### FIXED
```solidity
contract FixedLoanTransfer {
    mapping(uint256 => address) public loanLender;
    mapping(address => bool) public approvedLenders;

    function buyLoan(uint256 loanId, address newLender) external {
        require(
            approvedLenders[newLender] || msg.sender == newLender,
            "Lender not approved"
        );
        loanLender[loanId] = newLender;
    }

    function approveLender(address lender, bool approved) external {
        approvedLenders[lender] = approved;
    }
}
```

## Pattern #10: Loan State Manipulation via Refinancing

### VULNERABLE
```solidity
contract VulnerableRefinancing {
    enum LoanState { Active, Auction, Defaulted }
    mapping(uint256 => LoanState) public loanState;

    // ISSUE: Can refinance during auction to cancel it
    function refinance(uint256 loanId) external {
        require(loanState[loanId] != LoanState.Defaulted, "Defaulted");
        // Allows refinancing during auction
        loanState[loanId] = LoanState.Active;
        // Auction cancelled, loan extended indefinitely
    }
}
```

### FIXED
```solidity
contract FixedRefinancing {
    enum LoanState { Active, Auction, Defaulted }
    mapping(uint256 => LoanState) public loanState;

    function refinance(uint256 loanId) external {
        LoanState state = loanState[loanId];
        require(
            state == LoanState.Active,
            "Cannot refinance auctioned/defaulted loan"
        );

        // Additional checks
        require(meetsRefinancingCriteria(loanId), "Does not meet criteria");

        // Process refinancing
        loanState[loanId] = LoanState.Active;
    }
}
```

## Pattern #11: Double Debt Subtraction

### VULNERABLE
```solidity
contract VulnerableDebtAccounting {
    uint256 public poolBalance;
    mapping(uint256 => uint256) public loanDebt;

    function refinance(uint256 loanId) external {
        uint256 oldDebt = loanDebt[loanId];

        // ISSUE: Subtracts debt twice
        poolBalance -= oldDebt; // First subtraction

        uint256 newDebt = calculateNewDebt(loanId);
        loanDebt[loanId] = newDebt;

        // Later in the function...
        poolBalance -= oldDebt; // Second subtraction - corrupts balance!
        poolBalance += newDebt;
    }
}
```

### FIXED
```solidity
contract FixedDebtAccounting {
    uint256 public poolBalance;
    mapping(uint256 => uint256) public loanDebt;

    function refinance(uint256 loanId) external {
        uint256 oldDebt = loanDebt[loanId];
        uint256 newDebt = calculateNewDebt(loanId);

        // Atomic update: difference applied once
        if (newDebt > oldDebt) {
            poolBalance += (newDebt - oldDebt);
        } else {
            poolBalance -= (oldDebt - newDebt);
        }

        loanDebt[loanId] = newDebt;
    }
}
```

## Pattern #12: Dust Loan Griefing

### VULNERABLE
```solidity
contract VulnerableDustLoans {
    uint256 public minLoanSize = 1000e18;

    function createLoan(uint256 amount) external {
        require(amount >= minLoanSize, "Too small");
        // Create loan
    }

    // ISSUE: No minimum check on refinancing
    function refinance(uint256 loanId, uint256 newAmount) external {
        // Attacker refinances to dust amount, bypassing minimum
        loans[loanId].amount = newAmount;
    }
}
```

### FIXED
```solidity
contract FixedDustLoans {
    uint256 public minLoanSize = 1000e18;

    function createLoan(uint256 amount) external {
        require(amount >= minLoanSize, "Too small");
        // Create loan
    }

    function refinance(uint256 loanId, uint256 newAmount) external {
        // Minimum enforced on all operations
        require(newAmount >= minLoanSize, "Too small");
        loans[loanId].amount = newAmount;
    }

    function partialRepay(uint256 loanId, uint256 amount) external {
        uint256 remaining = loans[loanId].amount - amount;
        // Prevent leaving dust
        require(
            remaining == 0 || remaining >= minLoanSize,
            "Would leave dust amount"
        );
        loans[loanId].amount = remaining;
    }
}
```

## Summary: Key Protections

1. **Liquidation timing:** `paymentDueDate + gracePeriod`
2. **Collateral locks:** Cannot modify with active loan
3. **Closure validation:** Requires full repayment check
4. **Symmetric pauses:** Both repayment and liquidation
5. **Token restrictions:** New loans only
6. **Grace periods:** After unpause/re-allow
7. **Liquidation shares:** From total debt
8. **Address validation:** No zero addresses
9. **Loan transfers:** Require consent
10. **State constraints:** No refinancing during auctions
11. **Atomic accounting:** Single debt update
12. **Minimum enforcement:** All loan operations
