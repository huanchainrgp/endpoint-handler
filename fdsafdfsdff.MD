# Ludo Sát Phạt - Backend Implementation Plan

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Module Structure](#module-structure)
3. [Database Schema](#database-schema)
4. [Implementation Phases](#implementation-phases)
5. [Domain Models](#domain-models)
6. [WebSocket Events](#websocket-events)
7. [API Endpoints](#api-endpoints)
8. [Game Logic Flow](#game-logic-flow)

---

## Architecture Overview

Following the existing DDD 4-layer architecture:
- **Domain Layer**: Game entities, business rules, repository interfaces
- **Application Layer**: Use cases, DTOs
- **Infrastructure Layer**: Repository implementations, database models, migrations
- **Interface Layer**: HTTP handlers, WebSocket handlers, routes

**Key Technologies:**
- Gin Framework (HTTP)
- WebSocket (Gorilla WebSocket) for real-time game communication
- PostgreSQL for persistence
- Redis for caching game state
- Queue system (Asynq) for async operations

---

## Module Structure

### New Module: `internal/modules/game`

```
game/
├── domain/
│   ├── entities/
│   │   ├── game.go              # Game entity (room-based game instance)
│   │   ├── player.go            # Player entity (game participant)
│   │   ├── piece.go             # Piece entity (2 pieces per player)
│   │   ├── dice.go              # Dice entity (2 dice)
│   │   └── game_action.go       # Action entity (move, kill, etc.)
│   ├── repositories/
│   │   ├── game.go              # Game repository interface
│   │   ├── player.go            # Player repository interface
│   │   └── game_action.go       # Game action repository interface
│   ├── constants/
│   │   ├── errors.go            # Domain errors
│   │   ├── status.go            # Game status constants
│   │   ├── actions.go           # Action type constants
│   │   └── game_config.go       # Game configuration constants
│   └── value_objects/
│       ├── position.go          # Board position value object
│       ├── dice_result.go       # Dice roll result
│       └── board_path.go        # Board path calculation
├── application/
│   ├── dtos/
│   │   └── dto.go               # Game DTOs
│   └── use-cases/
│       ├── create_game.go       # Create game from room
│       ├── start_game.go        # Start game (start timer)
│       ├── roll_dice.go         # Roll dice use case
│       ├── move_piece.go        # Move piece use case
│       ├── handle_kill.go       # Handle kill & loot logic
│       ├── end_game.go          # End game (quick win or timed win)
│       ├── get_game_state.go    # Get current game state
│       └── check_win_condition.go # Check win conditions
├── infrastructure/
│   ├── persistence/
│   │   ├── migrations/
│   │   │   ├── 0001_create_games.up.sql
│   │   │   ├── 0001_create_games.down.sql
│   │   │   ├── 0002_create_game_players.up.sql
│   │   │   ├── 0002_create_game_players.down.sql
│   │   │   ├── 0003_create_game_pieces.up.sql
│   │   │   ├── 0003_create_game_pieces.down.sql
│   │   │   ├── 0004_create_game_actions.up.sql
│   │   │   └── 0004_create_game_actions.down.sql
│   │   ├── models/
│   │   │   ├── game.go
│   │   │   ├── player.go
│   │   │   ├── piece.go
│   │   │   └── game_action.go
│   │   └── repositories/
│   │       ├── game.go
│   │       ├── player.go
│   │       └── game_action.go
│   └── cache/
│       └── redis_game_state.go  # Redis cache for game state
└── interfaces/
    ├── http/
    │   ├── handler.go           # HTTP handlers
    │   └── routes.go            # HTTP routes
    └── websocket/
        ├── handler.go           # WebSocket handler
        ├── connection.go        # WebSocket connection wrapper
        └── events.go            # Event definitions
```

---

## Database Schema

### 1. `games` Table
```sql
CREATE TABLE games (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES rooms(id),
    status VARCHAR(20) NOT NULL DEFAULT 'waiting', -- waiting, playing, finished
    bet_amount BIGINT NOT NULL, -- Fixed bet amount per player
    current_player_id UUID, -- Current player's turn
    timer_started_at TIMESTAMP, -- When 5-minute timer started
    timer_duration_seconds INT NOT NULL DEFAULT 300, -- 5 minutes
    winner_id UUID, -- Winner player ID
    total_loot_amount BIGINT DEFAULT 0, -- Total loot collected by players
    house_fee_percent INT NOT NULL DEFAULT 5, -- House fee percentage (xâu)
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_games_room_id (room_id),
    INDEX idx_games_status (status),
    INDEX idx_games_timer (timer_started_at)
);
```

### 2. `game_players` Table
```sql
CREATE TABLE game_players (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_id UUID NOT NULL REFERENCES games(id),
    user_id UUID NOT NULL REFERENCES users(id),
    seat_index INT NOT NULL, -- 1-4
    position INT NOT NULL DEFAULT 0, -- Farthest piece position
    loot_amount BIGINT NOT NULL DEFAULT 0, -- Total loot collected
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(game_id, user_id),
    UNIQUE(game_id, seat_index),
    INDEX idx_game_players_game_id (game_id),
    INDEX idx_game_players_user_id (user_id)
);
```

### 3. `game_pieces` Table
```sql
CREATE TABLE game_pieces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_id UUID NOT NULL REFERENCES games(id),
    player_id UUID NOT NULL REFERENCES game_players(id),
    piece_number INT NOT NULL, -- 1 or 2 (only 2 pieces per player)
    position INT NOT NULL DEFAULT -1, -- -1 = home, 0-57 = board position, 58+ = finished area
    is_home BOOLEAN NOT NULL DEFAULT TRUE,
    is_finished BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(game_id, player_id, piece_number),
    INDEX idx_game_pieces_game_id (game_id),
    INDEX idx_game_pieces_player_id (player_id),
    INDEX idx_game_pieces_position (game_id, position)
);
```

### 4. `game_actions` Table
```sql
CREATE TABLE game_actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_id UUID NOT NULL REFERENCES games(id),
    player_id UUID NOT NULL REFERENCES game_players(id),
    action_type VARCHAR(20) NOT NULL, -- roll_dice, move_piece, kill, exit_home, finish
    dice_value_1 INT, -- First dice value (1-6)
    dice_value_2 INT, -- Second dice value (1-6)
    dice_sum INT, -- Sum of both dice
    is_doubles BOOLEAN NOT NULL DEFAULT FALSE, -- True if both dice same value
    piece_id UUID REFERENCES game_pieces(id),
    from_position INT,
    to_position INT,
    killed_piece_id UUID REFERENCES game_pieces(id), -- If this action killed a piece
    killed_player_id UUID REFERENCES game_players(id), -- If this action killed opponent
    loot_amount BIGINT DEFAULT 0, -- Loot amount if kill occurred
    sequence_number INT NOT NULL, -- Order of actions in the game
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_game_actions_game_id (game_id),
    INDEX idx_game_actions_player_id (player_id),
    INDEX idx_game_actions_sequence (game_id, sequence_number)
);
```

---

## Domain Models

### Game Entity
```go
type Game struct {
    ID                  string
    RoomID              string
    Status              GameStatus // waiting, playing, finished
    BetAmount           int64      // Fixed bet per player
    CurrentPlayerID     *string    // Current player's turn
    TimerStartedAt      *time.Time // When timer started
    TimerDuration       int        // Seconds (300 = 5 minutes)
    WinnerID            *string
    TotalLootAmount     int64
    HouseFeePercent     int        // Default 5%
    CreatedAt           time.Time
    UpdatedAt           time.Time
}

// Methods:
// - CanStart() error
// - Start() error
// - CanRollDice(playerID string) error
// - NextTurn() error
// - CheckWinCondition() (*Player, WinType)
// - IsTimerExpired() bool
// - GetTimeRemaining() time.Duration
```

### Player Entity
```go
type Player struct {
    ID          string
    GameID      string
    UserID      string
    SeatIndex   int    // 1-4
    Position    int    // Farthest piece position
    LootAmount  int64  // Total loot collected
    IsActive    bool
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// Methods:
// - UpdatePosition(newPosition int)
// - AddLoot(amount int64)
// - GetFarthestPiecePosition() int
```

### Piece Entity
```go
type Piece struct {
    ID          string
    GameID      string
    PlayerID    string
    PieceNumber int    // 1 or 2
    Position    int    // -1 = home, 0-57 = board, 58+ = finished
    IsHome      bool
    IsFinished  bool
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// Methods:
// - CanExitHome(diceSum int, isDoubles bool) bool
// - CanMove(diceSum int, currentPosition int) bool
// - Move(newPosition int) error
// - ExitHome() error
// - Finish() error
// - GetDistanceToFinish() int
```

### Dice Entity
```go
type DiceResult struct {
    Value1   int    // 1-6
    Value2   int    // 1-6
    Sum      int    // Sum of both dice
    IsDoubles bool  // True if both dice same value
}

// Methods:
// - Roll() DiceResult
// - CanExitHome() bool // Sum == 6 OR IsDoubles
```

---

## WebSocket Events

### Client → Server Events
```
game:join              - Join game room via WebSocket
game:roll_dice         - Roll dice
game:move_piece        - Move piece after dice roll
game:exit_home         - Exit piece from home
game:ping              - Keep-alive ping
```

### Server → Client Events
```
game:joined            - Successfully joined game
game:player_joined     - Another player joined
game:game_started      - Game has started (timer begins)
game:dice_rolled       - Dice roll result
game:piece_moved       - Piece moved
game:kill_occurred     - Kill & loot event
game:turn_changed      - Turn changed to next player
game:win_quick         - Quick win (piece reached finish)
game:win_timed         - Timed win (timer expired)
game:game_ended        - Game ended
game:timer_update      - Timer countdown update (every second)
game:error             - Error occurred
game:notification      - General notification
```

---

## API Endpoints

### HTTP REST API

#### Game Management
```
POST   /api/games                    - Create game from room (room must have 4 players)
GET    /api/games/:id                - Get game state
GET    /api/games/:id/players        - Get game players
GET    /api/games/:id/history        - Get game action history
POST   /api/games/:id/start          - Start game (only if 4 players)
```

#### WebSocket
```
WS     /ws/game/:gameId              - WebSocket connection for game
```

---

## Implementation Phases

### Phase 1: Core Game Setup (EPIC 1.1)
**Goal**: Players can create and join games

**Tasks:**
1. ✅ Create `game` module structure
2. ✅ Create database migrations for games, players, pieces, actions
3. ✅ Implement domain entities (Game, Player, Piece)
4. ✅ Implement repositories (interfaces + implementations)
5. ✅ Implement `CreateGame` use case (from room with 4 players)
6. ✅ Implement `GetGameState` use case
7. ✅ Implement HTTP handlers for game creation
8. ✅ Implement WebSocket connection handler

**Deliverables:**
- Games can be created from rooms
- Players can connect via WebSocket
- Game state can be retrieved

---

### Phase 2: Game Timer & Dice (EPIC 1.1)
**Goal**: Timer starts and dice rolling works

**Tasks:**
1. ✅ Implement timer service (Redis-based or in-memory)
2. ✅ Implement `StartGame` use case (starts 5-minute timer)
3. ✅ Implement `RollDice` use case
4. ✅ Implement dice domain logic (2 dice, sum calculation)
5. ✅ Broadcast timer updates via WebSocket
6. ✅ Handle timer expiration

**Deliverables:**
- Timer starts when game begins
- Players can roll dice on their turn
- Timer countdown broadcasted every second

---

### Phase 3: Piece Movement & Board Logic (EPIC 1.3)
**Goal**: Pieces can move on board

**Tasks:**
1. ✅ Implement board path calculation (value object)
2. ✅ Implement `ExitHome` use case (sum == 6 OR doubles)
3. ✅ Implement `MovePiece` use case
4. ✅ Implement position calculation logic
5. ✅ Handle "finish" logic (any roll >= remaining distance)
6. ✅ Broadcast piece movements via WebSocket

**Deliverables:**
- Pieces can exit home (sum == 6 OR doubles)
- Pieces can move based on dice sum
- Pieces can finish when close to goal

---

### Phase 4: Kill & Loot Basic (EPIC 2.1)
**Goal**: Basic kill and loot mechanics

**Tasks:**
1. ✅ Implement collision detection (landing on opponent)
2. ✅ Implement `HandleKill` use case
3. ✅ Implement loot calculation (10% of bet amount)
4. ✅ Integrate with wallet module for loot transactions
5. ✅ Broadcast kill events via WebSocket

**Deliverables:**
- Landing on opponent triggers kill
- 10% of bet is looted from victim
- Wallet transactions are created for loot

---

### Phase 5: Advanced Kill Mechanics (EPIC 2.2)
**Goal**: Piercing hunt and spawn kill

**Tasks:**
1. ✅ Implement path calculation with opponents
2. ✅ Implement "Piercing Hunt" (first opponent on path)
3. ✅ Implement "Landing Kill" (exact position)
4. ✅ Handle both in same turn
5. ✅ Implement jump-over logic (skip other opponents)
6. ✅ Implement spawn kill (at starting position)

**Deliverables:**
- Piercing hunt works (first opponent on path)
- Landing kill works
- Both can happen in same turn
- Spawn kill works

---

### Phase 6: Win Conditions (EPIC 1.2)
**Goal**: Quick win and timed win logic

**Tasks:**
1. ✅ Implement `CheckWinCondition` use case
2. ✅ Implement Quick Win (first piece to finish)
3. ✅ Implement Timed Win (farthest piece when timer expires)
4. ✅ Implement tie-breaker (most loot if same position)
5. ✅ Implement `EndGame` use case
6. ✅ Calculate winner payout ("Nhất Ăn Tất")
7. ✅ Integrate with wallet for payouts
8. ✅ Broadcast win events

**Deliverables:**
- Quick win triggers immediately
- Timed win triggers after 5 minutes
- Tie-breaker uses loot amount
- Winner receives all bets (minus house fee)

---

### Phase 7: Edge Cases & Polish (EPIC 2.3, 3.1, 3.2)
**Goal**: Handle edge cases and game rules

**Tasks:**
1. ✅ Handle own pieces (jump over, no kill)
2. ✅ Verify exit home rules (sum == 6 OR doubles)
3. ✅ Verify finish rules (any roll >= remaining distance)
4. ✅ Handle disconnections during game
5. ✅ Handle reconnections
6. ✅ Add game state persistence (Redis cache)
7. ✅ Add action history logging

**Deliverables:**
- All edge cases handled
- Game rules properly enforced
- Disconnections handled gracefully

---

## Game Logic Flow

### Game Start Flow
```
1. Room has 4 players → Create game
2. All players join via WebSocket
3. Owner/admin starts game → StartGame use case
4. Timer starts (5 minutes)
5. First player's turn begins
6. Broadcast game_started event
```

### Turn Flow
```
1. Current player's turn
2. Player rolls dice (2 dice, calculate sum)
3. If sum == 6 OR doubles → Can exit home
4. Player chooses action:
   - Exit home (if available)
   - Move piece (if pieces on board)
   - Skip turn (if no valid moves)
5. Check for kills:
   - Landing kill (exact position)
   - Piercing hunt (first opponent on path)
6. Execute kill & loot (if any)
7. Check win condition:
   - If piece finished → Quick Win
8. Move to next player's turn
9. Broadcast all events
```

### Win Condition Check
```
Quick Win:
- If any player has piece at finish → Win immediately
- Calculate payout: (bet_amount * 3) - house_fee

Timed Win (after 5 minutes):
- Find player with farthest piece position
- If tie → Player with most loot wins
- Calculate payout: (bet_amount * 3) - house_fee
```

### Kill & Loot Flow
```
1. Detect collision:
   - Landing kill: piece lands exactly on opponent
   - Piercing hunt: piece passes through opponent (first one)
2. Calculate loot: bet_amount * 10% (rounded down)
3. Create wallet transaction:
   - Debit from victim
   - Credit to attacker
4. Update player loot_amount
5. Move killed piece back to home
6. Broadcast kill event with details
```

---

## Integration Points

### Wallet Module Integration
- Use existing `CreditBalanceService` and `DebitBalanceService`
- Create transactions with reference_id = game_id
- Transaction metadata: `{"game_id": "...", "action": "loot", "victim_id": "..."}`

### Room Module Integration
- Game is created from room (room_id foreign key)
- Room status can be updated when game starts/finishes
- UserRoom records link to game players

### WebSocket Hub Integration
- Use existing `socket.Hub` for connection management
- Register connections by game_id (room_id)
- Broadcast to all players in game room

---

## Technical Considerations

### Performance
- Cache game state in Redis (TTL = game duration + buffer)
- Use Redis for timer (pub/sub for expiration events)
- Batch WebSocket broadcasts when possible

### Concurrency
- Use database transactions for game state updates
- Use Redis locks for turn management
- Handle concurrent dice rolls (prevent multiple rolls)

### Scalability
- Game state in Redis (fast access)
- Action history in PostgreSQL (audit trail)
- WebSocket connections per game room

### Error Handling
- Invalid moves → Return error via WebSocket
- Disconnections → Pause game or auto-skip turn
- Timer expiration → Auto-end game with timed win

---

## Testing Strategy

### Unit Tests
- Domain entities (Game, Player, Piece)
- Business rules (win conditions, kill logic)
- Value objects (position, dice result)

### Integration Tests
- Use cases (complete flows)
- Repository implementations
- Wallet integration

### E2E Tests
- Full game flow (4 players)
- WebSocket communication
- Timer expiration
- Win conditions

---

## Security Considerations

### Authentication
- JWT token required for HTTP endpoints
- WebSocket connection authenticated via token
- Verify player belongs to game room

### Authorization
- Only game participants can access game
- Only current player can roll dice/move
- Room owner can start game (if needed)

### Validation
- Validate all moves (position calculations)
- Validate dice rolls (prevent cheating)
- Validate win conditions

---

## Monitoring & Logging

### Key Metrics
- Active games count
- Average game duration
- Kill/loot frequency
- Timer expiration rate
- WebSocket connection stability

### Logging
- All game actions (audit trail)
- Kill events (for analytics)
- Win events (for analytics)
- Errors and exceptions

---

## Future Enhancements (Post-MVP)

1. Game replay system (from action history)
2. Leaderboards (most loot, most wins)
3. Tournament mode
4. Spectator mode
5. Betting variations (different bet amounts)
6. Power-ups (special moves)
7. Sound/visual effects configuration

---

## Next Steps

1. Review and approve this plan
2. Set up development environment
3. Begin Phase 1 implementation
4. Set up database migrations
5. Create initial module structure
6. Implement basic game creation flow
