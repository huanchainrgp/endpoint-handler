# Ludo Sát Phạt - Sprint Plan

## Sprint Structure Overview

```
Initiative: Ludo Sát Phạt Game Development
├── Epic 1: Core Gameplay Loop
│   ├── Feature 1.1: Game Setup & Timer
│   ├── Feature 1.2: Win Conditions
│   └── Feature 1.3: Payout System
├── Epic 2: Kill & Loot Mechanics
│   ├── Feature 2.1: Basic Kill & Loot
│   ├── Feature 2.2: Advanced Kill Mechanics
│   └── Feature 2.3: Edge Cases
└── Epic 3: Game Rules & Balancing
    ├── Feature 3.1: Exit & Entry Rules
    ├── Feature 3.2: Movement & Finish Rules
    └── Feature 3.3: Game Feel & Polish
```

---

## EPIC 1: Core Gameplay Loop
**Goal**: Players can complete a full game from start to finish

### Feature 1.1: Game Setup & Timer

#### Story 1.1.1: Create Game from Room
**Priority**: High | **Story Points**: 8

**Tasks**:
- [ ] **Subtask 1.1.1.1**: Create database migrations
  - Create `0001_create_games.up.sql` and `.down.sql`
  - Create `0002_create_game_players.up.sql` and `.down.sql`
  - Create `0003_create_game_pieces.up.sql` and `.down.sql`
  
- [ ] **Subtask 1.1.1.2**: Implement domain entities
  - Create `game/domain/entities/game.go`
  - Create `game/domain/entities/player.go`
  - Create `game/domain/entities/piece.go`
  
- [ ] **Subtask 1.1.1.3**: Implement repository interfaces
  - Create `game/domain/repositories/game.go`
  - Create `game/domain/repositories/player.go`
  
- [ ] **Subtask 1.1.1.4**: Implement repository implementations
  - Create infrastructure models and repositories
  
- [ ] **Subtask 1.1.1.5**: Implement CreateGame use case
  - Validate room has 4 players
  - Create game, players, and pieces
  
- [ ] **Subtask 1.1.1.6**: Implement HTTP handler
  - Create `POST /api/games` endpoint

**Acceptance Criteria**:
- ✅ Game created from room with 4 players
- ✅ Each player has 2 pieces at home
- ✅ Game status is "waiting"

---

#### Story 1.1.2: Game Timer (5 Minutes)
**Priority**: High | **Story Points**: 5

**Tasks**:
- [ ] **Subtask 1.1.2.1**: Implement timer service (Redis-based)
- [ ] **Subtask 1.1.2.2**: Implement StartGame use case
- [ ] **Subtask 1.1.2.3**: Broadcast timer updates (every second)
- [ ] **Subtask 1.1.2.4**: Handle timer expiration
- [ ] **Subtask 1.1.2.5**: Implement `POST /api/games/:id/start` endpoint

**Acceptance Criteria**:
- ✅ 5-minute timer starts when game begins
- ✅ Timer countdown broadcasted every second
- ✅ Timer expiration triggers timed win

---

#### Story 1.1.3: Dice Rolling (2 Dice, Sum)
**Priority**: High | **Story Points**: 3

**Tasks**:
- [ ] **Subtask 1.1.3.1**: Implement Dice value object
- [ ] **Subtask 1.1.3.2**: Implement RollDice use case
- [ ] **Subtask 1.1.3.3**: Implement WebSocket handler for dice roll
- [ ] **Subtask 1.1.3.4**: Add dice result to game state DTO

**Acceptance Criteria**:
- ✅ Player can roll 2 dice on their turn
- ✅ Result includes: value1, value2, sum, isDoubles
- ✅ Sum used for movement (not split)

---

### Feature 1.2: Win Conditions

#### Story 1.2.1: Quick Win (Piece Reaches Finish)
**Priority**: High | **Story Points**: 5

**Tasks**:
- [ ] **Subtask 1.2.1.1**: Implement finish detection logic
- [ ] **Subtask 1.2.1.2**: Implement CheckWinCondition use case
- [ ] **Subtask 1.2.1.3**: Implement EndGame use case
- [ ] **Subtask 1.2.1.4**: Integrate win check in MovePiece flow

**Acceptance Criteria**:
- ✅ Quick win triggers when piece reaches finish
- ✅ Game ends immediately
- ✅ win_quick event broadcasted

---

#### Story 1.2.2: Timed Win (Farthest Piece)
**Priority**: High | **Story Points**: 5

**Tasks**:
- [ ] **Subtask 1.2.2.1**: Implement position tracking
- [ ] **Subtask 1.2.2.2**: Implement Timed Win detection
- [ ] **Subtask 1.2.2.3**: Implement tie-breaker (loot_amount)
- [ ] **Subtask 1.2.2.4**: Trigger on timer expiration

**Acceptance Criteria**:
- ✅ Timed win triggers when timer expires
- ✅ Farthest piece wins
- ✅ Tie-breaker uses loot_amount

---

### Feature 1.3: Payout System

#### Story 1.3.1: Winner Payout Calculation
**Priority**: High | **Story Points**: 5

**Tasks**:
- [ ] **Subtask 1.3.1.1**: Implement payout calculation (bet_amount * 3 - house_fee)
- [ ] **Subtask 1.3.1.2**: Integrate with wallet module (CreditBalanceService)
- [ ] **Subtask 1.3.1.3**: Deduct bets from losers (DebitBalanceService)
- [ ] **Subtask 1.3.1.4**: Implement in EndGame use case

**Acceptance Criteria**:
- ✅ Winner receives (bet_amount * 3) - house_fee
- ✅ Each loser loses bet_amount
- ✅ All transactions atomic

---

## EPIC 2: Kill & Loot Mechanics
**Goal**: Implement "chất" sát phạt mechanics

### Feature 2.1: Basic Kill & Loot

#### Story 2.1.1: Landing Kill (Exact Position)
**Priority**: High | **Story Points**: 8

**Tasks**:
- [ ] **Subtask 2.1.1.1**: Implement collision detection
- [ ] **Subtask 2.1.1.2**: Implement HandleKill use case
- [ ] **Subtask 2.1.1.3**: Integrate with wallet (10% loot)
- [ ] **Subtask 2.1.1.4**: Update player loot tracking
- [ ] **Subtask 2.1.1.5**: Broadcast kill event

**Acceptance Criteria**:
- ✅ Landing on opponent triggers kill
- ✅ 10% of bet looted from victim
- ✅ Victim piece returns to home

---

#### Story 2.1.2: Kill Visual/Audio Effects Data
**Priority**: Medium | **Story Points**: 2

**Tasks**:
- [ ] **Subtask 2.1.2.1**: Enhance kill event payload (effect_type, animation, sound)
- [ ] **Subtask 2.1.2.2**: Add effect metadata to broadcast

**Acceptance Criteria**:
- ✅ Kill events include effect metadata
- ✅ Frontend can trigger appropriate effects

---

### Feature 2.2: Advanced Kill Mechanics

#### Story 2.2.1: Piercing Hunt (First Opponent on Path)
**Priority**: High | **Story Points**: 8

**Tasks**:
- [ ] **Subtask 2.2.1.1**: Implement board path calculation
- [ ] **Subtask 2.2.1.2**: Implement piercing hunt detection
- [ ] **Subtask 2.2.1.3**: Implement jump-over logic
- [ ] **Subtask 2.2.1.4**: Integrate with MovePiece use case

**Acceptance Criteria**:
- ✅ Piercing hunt kills first opponent on path
- ✅ Movement continues after kill
- ✅ Only first opponent killed

---

#### Story 2.2.2: Both Landing & Piercing in Same Turn
**Priority**: Medium | **Story Points**: 5

**Tasks**:
- [ ] **Subtask 2.2.2.1**: Implement multiple kill detection
- [ ] **Subtask 2.2.2.2**: Handle multiple loot transactions
- [ ] **Subtask 2.2.2.3**: Broadcast multiple kill events

**Acceptance Criteria**:
- ✅ Can trigger both kills in same move
- ✅ Multiple loot transactions created

---

### Feature 2.3: Edge Cases

#### Story 2.3.1: Own Pieces (Jump Over, No Kill)
**Priority**: Medium | **Story Points**: 3

**Tasks**:
- [ ] **Subtask 2.3.1.1**: Implement own piece detection
- [ ] **Subtask 2.3.1.2**: Update path calculation

**Acceptance Criteria**:
- ✅ Own pieces don't trigger kills
- ✅ Can pass through own pieces

---

#### Story 2.3.2: Spawn Kill (Starting Position)
**Priority**: Low | **Story Points**: 2

**Tasks**:
- [ ] **Subtask 2.3.2.1**: Allow kill at starting position (0)
- [ ] **Subtask 2.3.2.2**: Update collision detection

**Acceptance Criteria**:
- ✅ Can kill opponent at starting position
- ✅ Spawn kill works same as landing kill

---

## EPIC 3: Game Rules & Balancing
**Goal**: Ensure smooth, fast gameplay

### Feature 3.1: Exit & Entry Rules

#### Story 3.1.1: Exit Home Rules (Sum == 6 OR Doubles)
**Priority**: High | **Story Points**: 3

**Tasks**:
- [ ] **Subtask 3.1.1.1**: Implement exit home validation
- [ ] **Subtask 3.1.1.2**: Implement ExitHome use case
- [ ] **Subtask 3.1.1.3**: Integrate with MovePiece flow

**Acceptance Criteria**:
- ✅ Can exit with sum == 6 OR doubles
- ✅ Piece moves to position 0

---

#### Story 3.1.2: Only 2 Pieces Per Player
**Priority**: Low | **Story Points**: 1

**Tasks**:
- [ ] **Subtask 3.1.2.1**: Verify piece creation (2 per player)

**Acceptance Criteria**:
- ✅ Each player has exactly 2 pieces

---

### Feature 3.2: Movement & Finish Rules

#### Story 3.2.1: Finish Rule (Any Roll >= Remaining Distance)
**Priority**: High | **Story Points**: 5

**Tasks**:
- [ ] **Subtask 3.2.1.1**: Calculate distance to finish
- [ ] **Subtask 3.2.1.2**: Implement finish validation
- [ ] **Subtask 3.2.1.3**: Implement finish logic
- [ ] **Subtask 3.2.1.4**: Remove bounce-back logic

**Acceptance Criteria**:
- ✅ Can finish if diceSum >= remaining distance
- ✅ No bounce-back logic

---

### Feature 3.3: Game Feel & Polish

#### Story 3.3.1: Timer Warning Effects
**Priority**: Medium | **Story Points**: 2

**Tasks**:
- [ ] **Subtask 3.3.1.1**: Add warning thresholds (60s, 30s, 10s)
- [ ] **Subtask 3.3.1.2**: Enhance timer_update event

**Acceptance Criteria**:
- ✅ Timer warnings at 60s, 30s, 10s
- ✅ Event includes warning_level

---

#### Story 3.3.2: Leader Display (Farthest Position)
**Priority**: Medium | **Story Points**: 2

**Tasks**:
- [ ] **Subtask 3.3.2.1**: Calculate leader in game state
- [ ] **Subtask 3.3.2.2**: Include leader in DTO
- [ ] **Subtask 3.3.2.3**: Broadcast leader updates

**Acceptance Criteria**:
- ✅ Game state includes current leader
- ✅ Leader_changed event on switch

---

## Sprint Breakdown

### Sprint 1 (Weeks 1-2): Foundation
**Stories**: 1.1.1 (8 pts), 1.1.3 (3 pts)  
**Total**: 11 story points

### Sprint 2 (Weeks 3-4): Timer & Movement
**Stories**: 1.1.2 (5 pts), 3.1.1 (3 pts), 3.2.1 (5 pts)  
**Total**: 13 story points

### Sprint 3 (Weeks 5-6): Win Conditions
**Stories**: 1.2.1 (5 pts), 1.2.2 (5 pts), 1.3.1 (5 pts)  
**Total**: 15 story points

### Sprint 4 (Weeks 7-8): Kill & Loot Basic
**Stories**: 2.1.1 (8 pts), 2.1.2 (2 pts), 2.3.1 (3 pts)  
**Total**: 13 story points

### Sprint 5 (Weeks 9-10): Advanced Kills
**Stories**: 2.2.1 (8 pts), 2.2.2 (5 pts), 2.3.2 (2 pts)  
**Total**: 15 story points

### Sprint 6 (Weeks 11-12): Polish
**Stories**: 3.1.2 (1 pt), 3.3.1 (2 pts), 3.3.2 (2 pts)  
**Total**: 5 story points

---

## Dependencies

**Critical Path**:
1. Sprint 1 → Sprint 2 (games needed for timers)
2. Sprint 2 → Sprint 3 (movement needed for win conditions)
3. Sprint 3 → Sprint 4 (win conditions before kills)
4. Sprint 4 → Sprint 5 (basic kills before advanced)

**Parallel Work**:
- Story 3.1.2 can be done in Sprint 1
- Stories 3.3.1, 3.3.2 can be done in Sprint 5

---

## Definition of Done

1. ✅ All subtasks completed
2. ✅ Code reviewed
3. ✅ Unit tests passing
4. ✅ Integration tests passing
5. ✅ Acceptance criteria met
6. ✅ Documentation updated
7. ✅ No critical bugs
8. ✅ Deployed to staging
