# Code Examples: State Validation Vulnerabilities

## Vulnerable Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VulnerableStateValidationExamples {
    address public owner;
    address public pendingOwner;
    mapping(uint256 => address) public loans;
    mapping(address => bool) public paused;
    uint256 public lastUnpause;

    // Pattern #1: Unchecked 2-Step Ownership Transfer
    function transferOwnership_VULNERABLE(address newOwner) external {
        require(msg.sender == owner, "Not owner");
        pendingOwner = newOwner; // No check if newOwner != address(0)
    }

    function acceptOwnership_VULNERABLE() external {
        // Missing check: does NOT verify pendingOwner was set by current owner
        // Attacker can call this when pendingOwner == address(0)
        owner = pendingOwner;
        pendingOwner = address(0);
        // Result: ownership bricked to address(0)
    }

    // Pattern #2: Unexpected Matching Inputs
    function swap_VULNERABLE(address tokenIn, address tokenOut, uint256 amountIn)
        external
        returns (uint256)
    {
        // Missing validation: tokenIn != tokenOut
        // If tokenIn == tokenOut, nonsensical swap occurs
        // Could drain pools or bypass checks
        _transferFrom(tokenIn, msg.sender, address(this), amountIn);
        uint256 amountOut = _calculateSwapOutput(tokenIn, tokenOut, amountIn);
        _transfer(tokenOut, msg.sender, amountOut);
        return amountOut;
    }

    function addLiquidity_VULNERABLE(address token0, address token1, uint256 amount0, uint256 amount1)
        external
    {
        // No check: token0 != token1
        // Allows single-token "liquidity" that breaks pool invariants
        _transferFrom(token0, msg.sender, address(this), amount0);
        _transferFrom(token1, msg.sender, address(this), amount1);
    }

    // Pattern #3: Unexpected Empty Inputs
    function batchProcess_VULNERABLE(address[] calldata users, uint256[] calldata amounts)
        external
    {
        // No validation: arrays can be empty
        // Empty arrays bypass all logic below
        for (uint256 i = 0; i < users.length; i++) {
            _process(users[i], amounts[i]);
        }
        // If arrays are empty, function succeeds doing nothing
        // Could bypass important state transitions
    }

    function setConfig_VULNERABLE(uint256[] calldata values) external {
        // No check: values.length > 0
        // Could "successfully" set nothing, leaving protocol in invalid state
        for (uint256 i = 0; i < values.length; i++) {
            _updateConfig(i, values[i]);
        }
    }

    // Pattern #4: Unchecked Return Values
    function withdraw_VULNERABLE(address token, uint256 amount) external {
        // IERC20(token).transfer() returns bool but not checked
        IERC20(token).transfer(msg.sender, amount);
        // If transfer fails (returns false), balance updated anyway
        _updateBalance(msg.sender, token, amount);
    }

    function safeTransfer_VULNERABLE(address token, address to, uint256 amount) internal {
        // Low-level call return not checked
        (bool success, ) = token.call(abi.encodeWithSignature("transfer(address,uint256)", to, amount));
        // Silent failure if success == false
    }

    // Pattern #5: Non-Existent ID Manipulation
    function getLoanOwner_VULNERABLE(uint256 loanId) external view returns (address) {
        // No check: loan exists
        // Returns address(0) for non-existent loans
        return loans[loanId];
    }

    function repayLoan_VULNERABLE(uint256 loanId) external {
        address loanOwner = loans[loanId]; // Could be address(0) if loan doesn't exist
        // No validation that loan exists
        _transferFrom(address(this), loanOwner, 100 ether); // Sends to address(0)!
        delete loans[loanId]; // Deletes non-existent entry
    }

    function closeLoan_VULNERABLE(uint256 loanId) external {
        // Non-existent loan returns address(0) from mapping
        // Decrementing activeLoans even though loan never existed
        delete loans[loanId];
        activeLoans--; // Could underflow or mark loan as "closed"
    }

    // Pattern #6: Missing Access Control
    function mintRebalancer_VULNERABLE(address to, uint256 amount) external {
        // NO ACCESS CONTROL!
        // Anyone can mint tokens
        _mint(to, amount);
    }

    function buyLoan_VULNERABLE(uint256 loanId, address newLender) external {
        // NO ACCESS CONTROL!
        // Anyone can force-assign loans to unwilling lenders
        loans[loanId] = newLender;
    }

    function setFeeRecipient_VULNERABLE(address newRecipient) external {
        // NO onlyOwner or similar modifier
        // Anyone can redirect protocol fees
        feeRecipient = newRecipient;
    }

    function emergencyWithdraw_VULNERABLE(address token, uint256 amount) external {
        // Critical function with no access control
        IERC20(token).transfer(msg.sender, amount);
    }

    // Pattern #7: Inconsistent Array Length Validation
    function batchTransfer_VULNERABLE(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        // NO LENGTH VALIDATION!
        // If arrays have different lengths, will fail or process incorrectly
        for (uint256 i = 0; i < recipients.length; i++) {
            _transfer(msg.sender, recipients[i], amounts[i]); // OOB if amounts.length < recipients.length
        }
    }

    function updatePrices_VULNERABLE(
        address[] calldata tokens,
        uint256[] calldata prices
    ) external {
        // Mismatched lengths cause incorrect price assignments
        for (uint256 i = 0; i < tokens.length; i++) {
            tokenPrices[tokens[i]] = prices[i]; // Could revert or assign wrong prices
        }
    }

    // Pattern #8: Improper Pause Mechanism
    bool public repaymentsPaused;
    bool public liquidationsPaused;

    function pauseRepayments_VULNERABLE() external {
        repaymentsPaused = true;
        // Does NOT pause liquidations!
        // Users can't repay but can still be liquidated (unfair)
    }

    function unpauseRepayments_VULNERABLE() external {
        repaymentsPaused = false;
        lastUnpause = block.timestamp;
        // No grace period - users immediately liquidatable
    }

    function liquidate_VULNERABLE(address borrower) external {
        require(!liquidationsPaused, "Liquidations paused");
        // Can liquidate even if repayments paused
        // Can liquidate immediately after unpause (no grace period)
        _liquidate(borrower);
    }

    // Helper functions (stubs)
    function _transferFrom(address token, address from, address to, uint256 amount) internal {}
    function _transfer(address token, address to, uint256 amount) internal {}
    function _calculateSwapOutput(address tokenIn, address tokenOut, uint256 amountIn) internal pure returns (uint256) { return amountIn; }
    function _process(address user, uint256 amount) internal {}
    function _updateConfig(uint256 index, uint256 value) internal {}
    function _updateBalance(address user, address token, uint256 amount) internal {}
    function _mint(address to, uint256 amount) internal {}
    function _liquidate(address borrower) internal {}
    mapping(address => uint256) public tokenPrices;
    address public feeRecipient;
    uint256 public activeLoans;
}

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}
```

## Fixed Examples

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract FixedStateValidationExamples is Ownable2Step, ReentrancyGuard {
    address public pendingOwner;
    mapping(uint256 => address) public loans;
    mapping(uint256 => bool) public loanExists;
    uint256 public lastUnpause;
    uint256 public constant GRACE_PERIOD = 1 hours;

    // Pattern #1 FIXED: Properly Checked 2-Step Ownership Transfer
    // Option A: Use OpenZeppelin Ownable2Step (recommended)
    // Already inherits secure 2-step ownership from Ownable2Step

    // Option B: Manual implementation with checks
    function transferOwnership_FIXED_MANUAL(address newOwner) external onlyOwner {
        require(newOwner != address(0), "New owner is zero address");
        pendingOwner = newOwner;
        emit OwnershipTransferInitiated(owner(), newOwner);
    }

    function acceptOwnership_FIXED_MANUAL() external {
        // CRITICAL: Verify caller is the intended pendingOwner
        require(msg.sender == pendingOwner, "Not pending owner");
        require(pendingOwner != address(0), "No pending owner");

        address oldOwner = owner();
        _transferOwnership(pendingOwner);
        pendingOwner = address(0);

        emit OwnershipTransferred(oldOwner, owner());
    }

    // Pattern #2 FIXED: Validate Inputs Are Not Identical
    function swap_FIXED(address tokenIn, address tokenOut, uint256 amountIn)
        external
        nonReentrant
        returns (uint256)
    {
        // VALIDATION: Ensure different tokens
        require(tokenIn != tokenOut, "Identical tokens");
        require(tokenIn != address(0) && tokenOut != address(0), "Zero address");

        _transferFrom(tokenIn, msg.sender, address(this), amountIn);
        uint256 amountOut = _calculateSwapOutput(tokenIn, tokenOut, amountIn);
        _transfer(tokenOut, msg.sender, amountOut);
        return amountOut;
    }

    function addLiquidity_FIXED(
        address token0,
        address token1,
        uint256 amount0,
        uint256 amount1
    ) external nonReentrant {
        // Validate tokens are different
        require(token0 != token1, "Identical tokens");
        require(token0 != address(0) && token1 != address(0), "Zero address");
        require(token0 < token1, "Unsorted tokens"); // Canonical ordering

        _transferFrom(token0, msg.sender, address(this), amount0);
        _transferFrom(token1, msg.sender, address(this), amount1);
    }

    // Pattern #3 FIXED: Validate Non-Empty Inputs
    function batchProcess_FIXED(
        address[] calldata users,
        uint256[] calldata amounts
    ) external nonReentrant {
        // VALIDATION: Arrays not empty
        require(users.length > 0, "Empty users array");
        require(amounts.length > 0, "Empty amounts array");
        require(users.length == amounts.length, "Length mismatch");

        for (uint256 i = 0; i < users.length; i++) {
            require(users[i] != address(0), "Zero address");
            require(amounts[i] > 0, "Zero amount");
            _process(users[i], amounts[i]);
        }
    }

    function setConfig_FIXED(uint256[] calldata values) external onlyOwner {
        // Validate non-empty
        require(values.length > 0, "Empty values array");
        require(values.length <= MAX_CONFIG_SIZE, "Array too large");

        for (uint256 i = 0; i < values.length; i++) {
            _updateConfig(i, values[i]);
        }
    }

    // Pattern #4 FIXED: Check Return Values
    function withdraw_FIXED(address token, uint256 amount) external nonReentrant {
        // CHECK RETURN VALUE!
        bool success = IERC20(token).transfer(msg.sender, amount);
        require(success, "Transfer failed");

        _updateBalance(msg.sender, token, amount);
    }

    function safeTransfer_FIXED(address token, address to, uint256 amount) internal {
        // Check low-level call success
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSignature("transfer(address,uint256)", to, amount)
        );
        require(success, "Transfer call failed");

        // Check return value if any
        if (data.length > 0) {
            require(abi.decode(data, (bool)), "Transfer returned false");
        }
    }

    // Pattern #5 FIXED: Validate ID Existence
    function getLoanOwner_FIXED(uint256 loanId) external view returns (address) {
        // VALIDATE: loan exists
        require(loanExists[loanId], "Loan does not exist");
        return loans[loanId];
    }

    function repayLoan_FIXED(uint256 loanId) external nonReentrant {
        // Validate loan exists BEFORE any operations
        require(loanExists[loanId], "Loan does not exist");
        address loanOwner = loans[loanId];
        require(loanOwner != address(0), "Invalid loan owner");

        _transferFrom(address(this), loanOwner, 100 ether);
        delete loans[loanId];
        loanExists[loanId] = false;
    }

    function closeLoan_FIXED(uint256 loanId) external {
        // Validate existence before decrementing
        require(loanExists[loanId], "Loan does not exist");

        delete loans[loanId];
        loanExists[loanId] = false;
        activeLoans--;
    }

    // Pattern #6 FIXED: Add Access Control
    function mintRebalancer_FIXED(address to, uint256 amount) external onlyOwner {
        // ACCESS CONTROL: onlyOwner modifier
        require(to != address(0), "Zero address");
        _mint(to, amount);
    }

    function buyLoan_FIXED(uint256 loanId, address newLender) external {
        // ACCESS CONTROL: only loan owner or approved
        require(loanExists[loanId], "Loan does not exist");
        require(msg.sender == loans[loanId] || isApproved[msg.sender], "Not authorized");
        require(newLender != address(0), "Zero address");

        loans[loanId] = newLender;
    }

    function setFeeRecipient_FIXED(address newRecipient) external onlyOwner {
        // ACCESS CONTROL: onlyOwner
        require(newRecipient != address(0), "Zero address");
        feeRecipient = newRecipient;
    }

    function emergencyWithdraw_FIXED(address token, uint256 amount) external onlyOwner {
        // ACCESS CONTROL: onlyOwner for critical operations
        require(token != address(0), "Zero address");
        bool success = IERC20(token).transfer(msg.sender, amount);
        require(success, "Transfer failed");
    }

    // Pattern #7 FIXED: Validate Array Length Consistency
    function batchTransfer_FIXED(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external nonReentrant {
        // VALIDATE: matching lengths
        require(recipients.length > 0, "Empty array");
        require(recipients.length == amounts.length, "Length mismatch");

        for (uint256 i = 0; i < recipients.length; i++) {
            require(recipients[i] != address(0), "Zero address");
            require(amounts[i] > 0, "Zero amount");
            _transfer(msg.sender, recipients[i], amounts[i]);
        }
    }

    function updatePrices_FIXED(
        address[] calldata tokens,
        uint256[] calldata prices
    ) external onlyOwner {
        // Validate matching lengths
        require(tokens.length > 0, "Empty array");
        require(tokens.length == prices.length, "Length mismatch");

        for (uint256 i = 0; i < tokens.length; i++) {
            require(tokens[i] != address(0), "Zero address");
            require(prices[i] > 0, "Zero price");
            tokenPrices[tokens[i]] = prices[i];
        }
    }

    // Pattern #8 FIXED: Synchronized Pause Mechanism with Grace Period
    bool public protocolPaused;

    function pauseProtocol_FIXED() external onlyOwner {
        protocolPaused = true;
        // Pause BOTH repayments AND liquidations together
        emit ProtocolPaused(block.timestamp);
    }

    function unpauseProtocol_FIXED() external onlyOwner {
        protocolPaused = false;
        lastUnpause = block.timestamp;
        // Grace period enforced in liquidate function
        emit ProtocolUnpaused(block.timestamp);
    }

    function liquidate_FIXED(address borrower) external nonReentrant {
        require(!protocolPaused, "Protocol paused");

        // GRACE PERIOD: Can't liquidate immediately after unpause
        require(
            block.timestamp >= lastUnpause + GRACE_PERIOD,
            "Grace period active"
        );

        require(borrower != address(0), "Zero address");
        require(_isLiquidatable(borrower), "Not liquidatable");

        _liquidate(borrower);
    }

    function repay_FIXED() external nonReentrant {
        // Repayment also blocked during pause (synchronized)
        require(!protocolPaused, "Protocol paused");
        // Repayment logic...
    }

    // Events
    event OwnershipTransferInitiated(address indexed previousOwner, address indexed newOwner);
    event ProtocolPaused(uint256 timestamp);
    event ProtocolUnpaused(uint256 timestamp);

    // Helper functions (stubs)
    function _transferFrom(address token, address from, address to, uint256 amount) internal {}
    function _transfer(address token, address to, uint256 amount) internal {}
    function _calculateSwapOutput(address tokenIn, address tokenOut, uint256 amountIn) internal pure returns (uint256) { return amountIn; }
    function _process(address user, uint256 amount) internal {}
    function _updateConfig(uint256 index, uint256 value) internal {}
    function _updateBalance(address user, address token, uint256 amount) internal {}
    function _mint(address to, uint256 amount) internal {}
    function _liquidate(address borrower) internal {}
    function _isLiquidatable(address borrower) internal view returns (bool) { return true; }

    mapping(address => uint256) public tokenPrices;
    address public feeRecipient;
    uint256 public activeLoans;
    mapping(address => bool) public isApproved;
    uint256 constant MAX_CONFIG_SIZE = 100;
}

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}
```
