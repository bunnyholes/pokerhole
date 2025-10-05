# ROADMAP - PokerHole Project Status & Next Steps

**Last Updated**: 2025-10-04
**Current Phase**: Phase 3 Complete
**Test Status**: 151/151 passing (100%)

---

## Current Status

### Phase 1: Core Texas Hold'em (Complete)
- Game logic (PRE_FLOP → FLOP → TURN → RIVER → SHOWDOWN)
- Player actions (FOLD, CHECK, CALL, RAISE, ALL_IN)
- Winner determination (HandEvaluator, 21 golden tests)
- WebSocket real-time communication
- Server Authority model
- Server-client protocol synchronization

### Phase 2: Advanced Features (Complete)
- Blinds system (Small/Big Blind, configurable)
- Side pots (ALL_IN scenarios, multiple pots)
- Turn timeout (30 seconds auto-FOLD, thread-safe)

### Phase 3: Split Pot (Complete)
- Tie detection (HandResult.compareTo)
- Multiple winners handling (WinnerResolver)
- Equal distribution (PotDistributor)
- Remainder chip handling (dealer button priority)

### Test Coverage
**Total**: 151 tests
- HandEvaluatorTest: 25 (Golden vectors)
- TexasHoldemIntegrationTest: 18
- BlindsTest: 7
- SidePotTest: 9
- TurnTimeoutTest: 7
- HandResultTest: 34
- PotDistributorTest: 24
- WinnerResolverTest: 14
- SplitPotIntegrationTest: 7
- Architecture: 4
- Others: 2

---

## Next Steps (Phase 4+)

### Option 1: Dealer Button Rotation

**Priority**: Medium
**Effort**: 0.5 days
**Status**: Not Started

**Current Issue**:
- Dealer button is fixed
- Blinds always at same positions

**Implementation**:
1. Add `GameRoom.endHand()` method
2. Rotate button: `dealerButtonPosition = (dealerButtonPosition + 1) % players.size()`
3. Apply new blind positions on next hand

**Tests Needed**:
- Button rotation verification
- Multi-hand sequences
- Player elimination handling

---

### Option 2: Hand History Persistence

**Priority**: Medium
**Effort**: 3-4 days
**Status**: Not Started

**Implementation**:
1. Event store for game events
2. Save full hand record on completion
3. API endpoint for history query
4. Replay functionality

**Schema**:
```sql
CREATE TABLE hand_history (
    hand_id UUID PRIMARY KEY,
    game_id UUID NOT NULL,
    started_at TIMESTAMP NOT NULL,
    ended_at TIMESTAMP,
    final_pot INTEGER,
    winner_id UUID,
    events JSONB NOT NULL
);
```

---

### Option 3: Tournament Mode

**Priority**: Low
**Effort**: 1 week
**Status**: Not Started

**Implementation**:
1. Blind level increase (double every 10 minutes)
2. Player elimination tracking
3. Final ranking determination
4. Prize distribution logic

---

### Option 4: Integration Tests & Production Readiness

**Priority**: High
**Effort**: 2-3 days
**Status**: Not Started

**Implementation**:
1. Server-client E2E tests
2. Docker image build
3. CI/CD pipeline setup
4. Monitoring (Prometheus, Grafana)
5. Performance benchmarks

---

## Known Limitations & Technical Debt

### 1. Dealer Button Fixed
**Issue**: Button doesn't rotate between hands
**Impact**: Unfair game flow, same players always in blinds
**Fix**: Option 1 (Dealer Button Rotation)

### 2. Hand History Not Persisted
**Issue**: Game records only in memory
**Impact**: No audit trail, can't replay hands
**Fix**: Option 2 (Hand History Persistence)

### 3. Reconnection Handling Incomplete
**Issue**: Player disconnection hard to recover
**Impact**: Poor user experience, game disruption
**Fix**: WebSocket reconnection logic + game state resync

### 4. No Tournament Support
**Issue**: Only cash game mode available
**Impact**: Limited game variety
**Fix**: Option 3 (Tournament Mode)

---

## Architecture Overview

### Server Structure
```
pokerhole-server/src/main/java/dev/xiyo/pokerhole/
├── core/domain/              # Pure domain logic (no Spring)
│   ├── game/                # Game aggregate
│   │   ├── vo/SidePot.java
│   │   └── event/
│   ├── player/             # Player aggregate
│   └── card/               # Card, Deck, HandEvaluator
│
├── adapter/in/             # Input adapters
│   └── websocket/
│       ├── GameWebSocketHandler.java
│       └── service/
│           ├── GameCommandService.java
│           └── TurnTimeoutService.java
│
└── dealer/
    └── Dealer.java         # Core game logic
```

### Key Files

**Dealer.java**:
- `startTexasHoldem()`: Game initialization
- `processPlayerAction()`: Action handling
- `progressToNextRound()`: Round progression
- `determineWinner()`: Winner determination
- `createSidePots()`: Side pot calculation

**GameCommandService.java**:
- WebSocket action processing
- Timeout management
- Game state broadcasting

**TurnTimeoutService.java**:
- 30-second timeout tracking
- ScheduledExecutorService
- Thread safety

---

## Quick Start

### Server
```bash
cd pokerhole-server
docker compose up -d          # Start PostgreSQL
./gradlew bootRun            # Run server (port 8080)
./gradlew test               # Run all tests (151/151)
```

### Client
```bash
cd pokerhole-cli
go run cmd/poker-client/main.go
go test ./...
```

---

## Testing

### Run All Tests
```bash
./gradlew test
# Expected: BUILD SUCCESSFUL, 151 tests completed, 0 failed
```

### Run Specific Tests
```bash
./gradlew test --tests "BlindsTest"
./gradlew test --tests "SidePotTest"
./gradlew test --tests "TurnTimeoutTest"
./gradlew test --tests "SplitPotIntegrationTest"
```

### Test Report
```bash
./gradlew test
open build/reports/tests/test/index.html
```

---

## Documentation

### Essential Reading
1. **CLAUDE.md** - LLM work instructions, rules, conventions
2. **docs/adr/** - Architecture Decision Records
   - ADR-001: Event Sourcing
   - ADR-002: Server Authority
   - ADR-005: Deterministic RNG

### Server
- **pokerhole-server/README.md** - Server details

### Client
- **pokerhole-cli/README.md** - Client details

---

## Recommended Next Steps

**Phase 4 suggested order**:

1. **Dealer Button Rotation** (0.5 days)
   - Quick win
   - Improves game flow

2. **Integration Tests & Production** (2-3 days)
   - Production readiness
   - Performance validation
   - CI/CD pipeline

3. **Hand History** (3-4 days, optional)
   - Audit/analysis features
   - Leverages event sourcing

---

## Checklist Before Starting

### Environment
- [ ] Java 23+ installed
- [ ] Docker running
- [ ] PostgreSQL container up
- [ ] All tests passing (151/151)

### Knowledge
- [ ] CLAUDE.md read (rules & conventions)
- [ ] ADR documents read (ADR-001, ADR-002)
- [ ] Texas Hold'em rules understood
- [ ] Hexagonal Architecture pattern understood

### Before Implementation
- [ ] Existing code patterns studied
- [ ] Tests written first (TDD)
- [ ] Domain model purity maintained (no Spring in domain)
- [ ] Event sourcing pattern followed
- [ ] Protocol changes synchronized with client

### After Completion
- [ ] All tests passing (regression check)
- [ ] ROADMAP.md updated
- [ ] Documentation updated
- [ ] Code reviewed (optional)
- [ ] Git commit

---

## Key Principles

1. **Hexagonal Architecture**: Domain has ZERO framework dependencies
2. **Event Sourcing**: All state changes are events (immutable)
3. **Server Authority**: Client proposes, server validates
4. **Test-First**: Write tests before implementation
5. **Documentation**: Keep ROADMAP.md current after each change
