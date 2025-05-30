# Issue H-1: Incorrect tier tracking when tier 3 staker exits in a certain case 

Source: https://github.com/sherlock-audit/2025-05-layeredge-judging/issues/83 

## Found by 
MysteryAuditor, X77, future, rudhra1749, slavina

### Summary

Incorrect tier tracking when tier 3 staker exits in a certain case

### Root Cause

```solidity
            else if (isRemoval) {
                _findAndRecordTierChange(new_t1 + new_t2, n);
            }
```
Whenever a staker exits and the old T2 and new T2 counts are the same, then we'd end up above. However, this code does not properly work when a T3 staker exits from the tree and completely breaks the tier tracking.

In our scenario below (attack path and POC), there are 7 T1 and T2 and 8 T3 stakers (15 total) stakers initially. T3 staker exits, so now there must be 2 T1 stakers, 4 T2 stakers and 8 T3 stakers where the last T2 staker is demoted. However, in the snippet shown above, we write to `new_t1 + new_t2` rank which is 6 (2 + 4). However, as a T3 staker exited, then the 7th rank is the actual last T2 staker, not the 6th we are writing to.

### Internal Pre-conditions

- Check in the root cause snippet should be reached, this happens when the T2 count before and after a removal are the same

### External Pre-conditions

-

### Attack Path

1. There are 15 stakers, distribution is:
- T1 = 15 * 0.2 = 3
- T2 = 15 * 0.3 = 4
- T3 = 15 - 3 - 4 = 8
2. User from T3 unstakes, distribution should be:
- T1 = 14 * 0.2 = 2
- T2 = 14 * 0.3 = 4
- T3 = 14- 4 - 2 = 8
3. The following code is reached upon the removal and the T2 boundary handling block:
```solidity
            else if (isRemoval) {
                _findAndRecordTierChange(new_t1 + new_t2, n);
            }
```
4. We write to the 6th rank (2 + 4), however as a T3 staker exited, then the current T1 and T2 staker counts are still the initial 7 (3 + 4), thus we must write to rank 7 (if T1 or T2 exits, then the 6th rank is properly demoted as that is the correct boundary as T1 and T2 current counts are now 6 due to the exit, however here, 7th is the correct boundary as T3 exited and T1 and T2 counts didn't change)
5. As we write to the 6th rank, he remains T2 instead of the 7th rank demoting to T3, thus the distribution breaks and is now `2 T1 (correct), 5 T2 (incorrect), 7 T3 (incorrect)`

### Impact

Incorrect tier tracking, loss of funds for the protocol as they must handle a bigger APY due to more users being in T2 unfairly

### PoC

First, due to issue starting with title `An edge case in ....`, we must do a little fix to reach the proper state:
```diff
            } else if (!isRemoval) {
+                _findAndRecordTierChange(new_t1 + new_t2, n);
-                _findAndRecordTierChange(old_t1 + old_t2, n);
            }
```
Note that I am not sure whether this fix is perfect, however it makes the state consistent, at least in the flow for our issue (we have assertions for that in our POC below).

Then, add this function in the staking contract to easily get the tier of a user:
```solidity
    function getTier(address user) public view returns (uint256) { // note: added
        if (stakerTierHistory[user].length == 0) return 0;
        TierEvent memory te = stakerTierHistory[user][stakerTierHistory[user].length - 1];
        return uint256(te.to);
    }
```

Now, paste the following POC in the staking test contract:
```solidity
       function testIssue() public {
        for (uint256 i; i < 15; i++) {
            address user = address(uint160(i + 1));
            deal(address(token), user, 3000e18);
            vm.startPrank(user);
            token.approve(address(staking), MIN_STAKE);
            staking.stake(MIN_STAKE);
        }

        uint256 t1Count;
        uint256 t2Count;
        uint256 t3Count;
        for (uint256 i; i < 15; i++) {
            address user = address(uint160(i + 1));
            uint256 tier = staking.getTier(user);
            if (tier == 1) {
                t1Count++;
                continue;
            }

            if (tier == 2) {
                t2Count++;
                continue;
            }

            if (tier == 3) {
                t3Count++;
                continue;
            }
        }

        // NOTE: Changed `old_t1 + old_t2` to `new_t1 + new_t2` in the last block of `_checkBoundariesAndRecord()` so we get into a correct state due to another issue (not sure if this is the proper fix but it works in this case)
        assertEq(staking.stakerCountInTree(), 15);
        (uint256 t1ShouldBe, uint256 t2ShouldBe, uint256 t3ShouldBe) = staking.getTierCountForStakerCount(staking.stakerCountInTree());

        assertEq(t1Count, 3);
        assertEq(t1ShouldBe, t1Count);
        assertEq(t2Count, 4);
        assertEq(t2ShouldBe, t2Count);
        assertEq(t3Count, 8);
        assertEq(t3ShouldBe, t3Count);

        address unstakingUser = address(uint160(10));
        uint256 r = staking.getTier(unstakingUser);
        assertEq(r, 3);

        vm.startPrank(unstakingUser);
        staking.unstake(MIN_STAKE);

        assertEq(staking.stakerCountInTree(), 14);
        (t1ShouldBe, t2ShouldBe, t3ShouldBe) = staking.getTierCountForStakerCount(staking.stakerCountInTree());

        t1Count = 0;
        t2Count = 0;
        t3Count = 0;
        for (uint256 i; i < 15; i++) {
            address user = address(uint160(i + 1));
            if (unstakingUser == user) continue; // He left the tree above so shouldn't include him
            uint256 tier = staking.getTier(user);
            if (tier == 1) {
                t1Count++;
                continue;
            }

            if (tier == 2) {
                t2Count++;
                continue;
            }

            if (tier == 3) {
                t3Count++;
                continue;
            }
        }

        assertEq(t1Count, 2); // Correct!
        assertEq(t1Count, t1ShouldBe);

        assertEq(t2Count, 5); // Incorrect!
        assertNotEq(t2Count, t2ShouldBe);

        assertEq(t3Count, 7); // Incorrect
        assertNotEq(t3Count, t3ShouldBe);
    }
```

### Mitigation

_No response_

# Issue H-2: Incorrect update of tier give permanent position to some users. 

Source: https://github.com/sherlock-audit/2025-05-layeredge-judging/issues/287 

## Found by 
Cybrid, Hecok, Hurricane, Pablo, X77, benjamin\_0923, durov, future, jkoppel, kazan, lukris02, montecristo, newspacexyz, rudhra1749, shiazinho, slavina, t0x1c, wickie

### Summary

The `LayerEdgeStaking.sol` contract uses the FCFS (First-Come-First-Served) model to give the earliest stakers the highest APYs. There are three tiers Tier 1, Tier 2 And Tier 3, with the distribution of 20%, 30% and 50% of stakers in each tier, and in order of Highest APY to Lowest. Users can only enter Tier 1 and Tier 2 if they at least stake the [`MIN_STAKE`](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L144) amount and they are among the first 20% or 30% of stakers. Everyone else will be in Tier 3 but only users who staked the `MIN_STAKE` amount can move up to Tier 2 or 1 from Tier 3 when users in front of them unstake. As users stake and unstake they're moved from tier to tier based on their rank(i.e how early they staked in respective of other users). This is done via `LayerEdgeStaking.sol::_checkBoundariesAndRecord()`, which is called as when users in the system stake or unstake. 

The function works fine when, 
    1. a user from Tier 3 move to Tier 2.    
    2. a user from Tier 3 move to Tier 2 AND another user move from Tier 2 to Tier 1.

However, when only a single user move, from Tier 2 to Tier 1, the function fails to fill the gap of Tier 2 with a user from Tier 3. This is corrected when a user stakes or unstakes (triggering `_checkBoundariesAndRecord()`), but this user stays in Tier 2 even when all the users who staked after him unstakes.

The _checkBoundariesAndRecord() is responsible for handling the boundries of Tiers and updating the users' tiers based on the number of stakers. 
Cases where,
1. Both Tier 1 and Tier 2 boundry changes
2. Tier 2 boundry changes, are handled correctly.

However, when ONLY Tier 1 boundry changes, the function updates the tiers of the users at the boundry incorrectly.
For example, based on the tier distribution of 20%, 30%, 50% in Tier 1, 2 and 3 respectively,

1. At 14 users, Tier 1 HAS 2 users (U1, U2), Tier 2 HAS 4 users (U3, U4, U5, U6) and Tier 3 HAS 8 users (U7-U14).  
2. A user stakes and is put in Tier 3.
3. At 15 users, Tier 1 SHOULD HAVE 3 users (U1, U2, U3), Tier 2 SHOULD HAVE 4 users (U4, U5, U6, U7) and Tier 3 SHOULD HAVE 8 users (U8-U15).

But the function handles this case incorrectly and in reality, 
At 15 users, Tier 1 HAS 3 users (U1, U2, U3), Tier 2 HAS 3 users (U4, U5, U6) and Tier 3 HAS 9 users (U7, U8-U15).
This is because, the function handles this case in reverse. 

At the end of  [_checkBoundariesAndRecord()](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L894-L899), when handling tier 2 boundries, the removal cases are handled in reverse.  

```solidity
    function _checkBoundariesAndRecord(bool isRemoval) internal {
        ....
            //Handle case where Tier 2 count stays the same
            else if (isRemoval) {
                _findAndRecordTierChange(new_t1 + new_t2, n);
@>          } else if (!isRemoval) {
                _findAndRecordTierChange(old_t1 + old_t2, n);
            }
        }
    }
```

So in the example above, isRemoval == false, new_t1 = 3, new_t2 = 4, old_t1 = 2, old_t2 = 4 and n = 15.
The [_findAndRecordTierChange()](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L907) is called with `6` as the input `rank`. This means, the rank change is excuted on U6, but he is already in Tier 2 so the _recordTierChange() does nothing. 
In reality, the rank change should be executed on U7, who moved to Tier 2 from Tier 3, because there is a gap in Tier 2, caused by U3 moving to Tier 1.

```solidity
    function _findAndRecordTierChange(uint256 rank, uint256 _stakerCountInTree) internal {
        uint256 joinIdCross = stakerTree.findByCumulativeFrequency(rank);
        address userCross = stakerAddress[joinIdCross];
        uint256 _rank = stakerTree.query(joinIdCross);
        Tier toTier = _computeTierByRank(_rank, _stakerCountInTree);
        _recordTierChange(userCross, toTier);
    }
```

The [_computeTierByRank()](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L921) finds the users Tier based on his rank and [_recordTierChange()](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L817) updates it.

```solidity 
    function _computeTierByRank(uint256 rank, uint256 totalStakers) internal pure returns (Tier) {
        if (rank == 0 || rank > totalStakers) return Tier.Tier3; 
        (uint256 tier1Count, uint256 tier2Count,) = getTierCountForStakerCount(totalStakers);
        if (rank <= tier1Count) return Tier.Tier1;
        else if (rank <= tier1Count + tier2Count) return Tier.Tier2;
        return Tier.Tier3;
    }

    function _recordTierChange(address user, Tier newTier) internal {
        Tier old = Tier.Tier3;
        if (stakerTierHistory[user].length > 0) {
            old = stakerTierHistory[user][stakerTierHistory[user].length - 1].to;
        }
        if (
            stakerTierHistory[user].length > 0
                && stakerTierHistory[user][stakerTierHistory[user].length - 1].to == newTier
        ) return;
        uint256 currentTime = block.timestamp;
        stakerTierHistory[user].push(TierEvent({from: old, to: newTier, timestamp: currentTime}));
        users[user].lastTimeTierChanged = currentTime;

        emit TierChanged(user, newTier);
    }
```

If a new staker enters, U7 is correctly placed in Tier 2.
At 16 users, Tier 1 HAS 3 users (U1, U2, U3), Tier 2 HAS 4 users (U4, U5, U6, U7) and Tier 3 HAS 9 users (U8-U16).

However, in this case, U7 gains a permanent position in Tier2. Even after U8-16 unstakes, he is still in Tier 2.
At 7 users, Tier 1 SHOULD HAVE 1 user(U1), Tier 2 SHOULD HAVE 2 users (U2, U3) Tier 3 SHOULD HAVE 4 users. (U4, U5, U6, U7)
In reality, Tier 1 HAS 1 user(U1), Tier 2 HAS 3 users (U2, U3, U7), Tier 3 HAS 3 users. (U4, U5, U6).

Interests are calculated and updated via [calculateUnclaimedInterest](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L397), which uses `stakerTierHistory`. 
If the user's history is wrong, (Tier 3 -> Tier 2, instead of Tier 3 -> Tier 2 -> Tier 3), they will get interests based on the wrong history.

### Root Cause

In [_checkBoundariesAndRecord()](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L894-L899), when handling tier 2 boundries, the removal cases are handled in reverse.  

```solidity
    function _checkBoundariesAndRecord(bool isRemoval) internal {
        ....
            //Handle case where Tier 2 count stays the same
            else if (isRemoval) {
                _findAndRecordTierChange(new_t1 + new_t2, n);
@>          } else if (!isRemoval) {
                _findAndRecordTierChange(old_t1 + old_t2, n);
            }
        }
    }
```

### Internal Pre-conditions

None

### External Pre-conditions

ONLY Tier 1's boundry should be increased. 

### Attack Path

None, logical error. 

### Impact

This occurs everytime when ONLY Tier 1 boundry is increased. If no new stakers enter, the user who should be in Tier 2 is stuck in Tier 3, missing out on Tier 2 APYs. However, if new stakers enters, This particular user gains permanent position in Tier 2. In our POC, Alice gets Tier 2 APY, when she should be gettin Tier 3 APY. 
This issue breaks the invariants

    Tier distribution correctness. 
Some users' tiers are not updated accordingly (users joining as 7th, 12th, 17th, 22th, 27th, ... staker). If there is no new staker to bump them up the ranking, they will get Tier 3 APY instead of Tier 2's. If they are bumped, they will get permanent Tier 2 APY even when they shold be gettin Tier 3's.

    First-come-first-serve (FCFS) ordering.
 Such users will unfairly get more APY and the user who staked before them will be demoted first before them.

### PoC

Paste this test in src/test/stake/TierBoundryAndInterestTest.t.sol.

Prerequisites: 
1. import OZ's [String.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/441b1c1c5b909498ca6206a84445ee516067e7fc/contracts/utils/Strings.sol#L13), this is used to create new users and stake.
2. Add this helper function, addUsersMinStake(), which creates new users and stake the `MIN_STAKE`

```solidity
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";

contract TierBoundaryAndInterestTest is Test {
    
    function addUsersMinStake(uint256 num) public {     
        for (uint256 i = 1; i < num; i++) { //starts from 1 so that first staker appears as `user1` in logs.
            address[] memory users = new address[](num);
            users[i] = makeAddr(Strings.toString(i));
            dealToken(users[i], MIN_STAKE);
            vm.prank(users[i]);
            staking.stake(MIN_STAKE); 
        }}
```

This is a test showing how alice the 7th staker(U7), is moved to Tier 2 only after a new user staked and stayed in Tier 2 althoough stakers after her unstaked. She is getting APY of TIer 2.

```solidity
    function test_permenantTier() public {
        addUsersMinStake(7); //Add 6 stakers in front of Alice 
        address[10] memory users = [alice, bob, charlie, dave, eve, frank, grace, heidi, ivan, judy];

        //Alice is the 7th staker
        for (uint256 i = 0; i < 9; i++) {
            dealToken(users[i], MIN_STAKE);
            vm.prank(users[i]);
            staking.stake(MIN_STAKE);
        }

        //15 stakers, so Alice(7th staker) SHOULD BE in Tier 2
        assertEq(staking.stakerCountInTree(), 15);

        //NOTE this returns the SHOULD BE tier. In reality Alice has NOT been moved to Tier 2.
        LayerEdgeStaking.Tier tier = staking.getCurrentTier(alice);
        assert(tier == LayerEdgeStaking.Tier.Tier2);

        //Alice history length is still 1, she has NOT been moved to Tier 1
        uint256 aliceHistory = staking.stakerTierHistoryLength(alice);
        assertEq(aliceHistory, 1);
        
        //Judy stakes, she is the 16th staker
        dealToken(judy, MIN_STAKE);
        vm.prank(judy);
        staking.stake(MIN_STAKE);

        //Alice history length is now 2, 1 from staking, one from Tier3 -> Tier2
        aliceHistory = staking.stakerTierHistoryLength(alice);
        console2.log("Alice History Length", aliceHistory);
        assertEq(aliceHistory, 2);

        //Every that stakes after alice unstakes. U16-U8
        for (uint256 i = 9; i > 0; i--) {
            vm.prank(users[i]);
            staking.unstake(MIN_STAKE);
        }
        //Alice is not demoted back to Tier3. With this history, she will get Tier2 APY. 
        aliceHistory = staking.stakerTierHistoryLength(alice);
        console2.log("Alice History Length", aliceHistory);
        assertEq(aliceHistory, 2);

        //Alice is getting Tier 2 APY
        skip(365 days); //1 year later, 35 % interest
        uint256 aliceInterest = staking.calculateUnclaimedInterest(alice);
        console2.log("Alice Interest", aliceInterest);
        assertEq(aliceInterest, (MIN_STAKE * 35) / 100);
    }
```

NOTE : the [getCurrentTier()](https://github.com/sherlock-audit/2025-05-layeredge/blob/main/edgen-staking/src/stake/LayerEdgeStaking.sol#L355) returns the correct Tier of the user despite their history. i.e The function will return Alice's Tier as 2 before Judy stakes, but in reality Alice still has a history length of 1, meaning her history has not yet been updated. This can be seen in the logs as well.
After all the users who staked after alice unstaked, she has not be demoted to Tier3 (historyLength == 2). If she has been demoted to Tier 3, she should have a history length of 3.

This is a POC showing the mitigation correctly updated Alice's history. Run this after applying the mitigation.

```solidity
    function test_permenantTierMitigation() public {
        addUsersMinStake(7); //Add 6 stakers in front of Alice 
        address[10] memory users = [alice, bob, charlie, dave, eve, frank, grace, heidi, ivan, judy];

        //Alice is the 7th staker
        for (uint256 i = 0; i < 9; i++) {
            dealToken(users[i], MIN_STAKE);
            vm.prank(users[i]);
            staking.stake(MIN_STAKE);
        }
        
        //15 stakers, so Alice(7th staker) SHOULD BE in Tier 2
        assertEq(staking.stakerCountInTree(), 15);
        LayerEdgeStaking.Tier tier = staking.getCurrentTier(alice);
        assert(tier == LayerEdgeStaking.Tier.Tier2);
        //Alice history length is 2, she has be promoted to Tier 2.
        uint256 aliceHistory = staking.stakerTierHistoryLength(alice);
        assertEq(aliceHistory, 2);
        
        //Alice was correctly promoted to Tier 2 when the 15th Staker(ivan) staked.
        //Judy stakes, she is the 16th staker
        dealToken(judy, MIN_STAKE);
        vm.prank(judy);
        staking.stake(MIN_STAKE);

        //Every that stakes after alice unstakes. U16-U8
        for (uint256 i = 9; i > 0; i--) {
            vm.prank(users[i]);
            staking.unstake(MIN_STAKE);
        }
        //Alice is demoted back to Tier3.
        aliceHistory = staking.stakerTierHistoryLength(alice);
        console2.log("Alice History Length", aliceHistory);
        assertEq(aliceHistory, 3);

        skip(365 days); //1 year later, 20% interest
        uint256 aliceInterest = staking.calculateUnclaimedInterest(alice);
        console2.log("Alice Interest", aliceInterest);
        assertEq(aliceInterest, (MIN_STAKE * 20) / 100);
    }
```

### Mitigation

Flip the conditions in _checkBoundariesAndRecord().

```diff
    function _checkBoundariesAndRecord(bool isRemoval) internal {
        ....
            //Handle case where Tier 2 count stays the same
-           else if (isRemoval) {
+           else if (!isRemoval) {
                _findAndRecordTierChange(new_t1 + new_t2, n);
-           } else if (!isRemoval) {
+           } else if (isRemoval) {
                _findAndRecordTierChange(old_t1 + old_t2, n);
            }
        }
    }
```


