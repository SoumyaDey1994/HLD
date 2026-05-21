## Major Flows

### 1. Update Score Flow
    Goal: user_1 gains +50 points
    Command: ZINCRBY leaderboard +50 user_1

    Step-by-Step Internal Flow
        - Step 1 → Leaderboard API Receives Score Update
            POST /score/update
            { userId: user_1, delta: +50 }

        - Step 2 → HashMap Finds Existing Score
            user_1 -> 9800, Lookup: O(1), Current Score: 9800

        - Step 3 → Compute New Score
            9800+50 = 9850

        - Step 4 → Remove Old Node From SkipList
            Old Node: (9800, user_1), SkipList traversal: O(logN)

        - Step 5 → Insert New Node Into Correct Position
            New Node: (9850, user_1), Redis inserts in sorted order, TC: O(logN)

        - Step 6 → Update HashMap
            user_1 -> 9850, TC: O(1)

    Final Complexity: O(log N) as SkipList repositioning dominates.

### 2. Get Rank Flow
    Goal: Find rank of user_1
    Command: ZREVRANK leaderboard user_1

    Step-by-Step Internal Flow
        - Step 1: Locate Member
        - Step 2: Find Node In SkipList
        - Step 3: Use Span Metadata
        - Step 4: Compute Final Rank

    Final Complexity: O(log N) because SkipList avoids full traversal.

### 3. Get Top N Users Flow
    Goal: Get top 100 players
    Command: ZREVRANGE leaderboard 0 99 WITHSCORES

    Step-by-Step Internal Flow:
        - Step 1: Jump To Highest Score Node [O(logN)]
        - Step 2: Sequential Traversal [O(M)] where M = Top M
        - Step 3: Return Result

    Final Complexity: O(log N + M) where M = returned users

### 4. Get Nearby Users Flow
    Goal: Show 5 users above & below user_1

    Step-by-Step Internal Flow:
        - Step 1: Find User Rank
            - ZREVRANK leaderboard user_1 --> O(log N)
        - Step 2: Compute Range
        - Step 3: Traverse Nearby Nodes (Linked Skiplist Traversal)
        - Step 4: Return Users
    
    Final Complexity: O(log N + M) where M = nearby returned users


## Final Summary

    | Flow         | Main Internal Mechanism            | Complexity   |
    | ------------ | ---------------------------------- | -------------|
    | Update Score | Remove + reinsert in SkipList      | O(log N)     |
    | Get Rank     | SkipList traversal + span counting | O(log N)     |
    | Get Top N    | Jump + sequential traversal        | O(log N + M) |
    | Nearby Users | Rank lookup + local traversal      | O(log N + M) |

## Core Insight

Redis Sorted Set succeeds because:
```
    HashMap  -> fast member lookup
    SkipList -> fast ordered traversal
```
This hybrid design makes Redis ideal for real-time leaderboard systems.