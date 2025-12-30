# Staking & Reward Vulnerability Patterns

## Pattern #1: Front-Running First Deposit
**Risk:** Attacker front-runs first deposit to become initial staker, then steals initial WETH rewards by sandwiching the deposit
**Detection:** Check if reward token can be same as staking token, or if initial deposit protection exists
**Impact:** First depositor steals all initial rewards meant for protocol, users receive nothing

## Pattern #2: Reward Dilution via Direct Transfer
**Risk:** Sending staking tokens directly to contract increases totalSupply without proper accounting, diluting rewards per share
**Detection:** Verify totalSupply tracks staked amounts separately from actual token balance
**Impact:** Attacker dilutes rewards for legitimate stakers, reducing their yield or stealing rewards

## Pattern #3: Precision Loss in Reward Calculation
**Risk:** Small stake amounts or frequent reward updates cause calculated rewards to round down to zero
**Detection:** Check reward calculation precision, minimum stake requirements, and scaling factors
**Impact:** Users with small stakes receive no rewards despite earning them, accumulated loss over time

## Pattern #4: Flash Deposit/Withdraw Griefing
**Risk:** Large instant deposit followed by immediate withdrawal dilutes rewards for existing stakers without committing capital
**Detection:** Verify time locks, minimum stake duration, or anti-sandwich mechanisms
**Impact:** Attacker repeatedly griefs stakers by diluting rewards in single block, reducing everyone's yield

## Pattern #5: Update Not Called After Reward Distribution
**Risk:** Adding new rewards doesn't update rewardPerToken index, causing stale values in subsequent calculations
**Detection:** Check if updateReward() or equivalent called before/after reward distribution
**Impact:** Rewards calculated incorrectly, users receive wrong amounts (too much or too little)

## Pattern #6: Balance Caching Issues
**Risk:** Claiming rewards updates cached user balance incorrectly, causing subsequent calculations to use stale values
**Detection:** Verify balance tracking during claim operations and state consistency
**Impact:** Users claim incorrect reward amounts, double-claim exploits, or reward loss
