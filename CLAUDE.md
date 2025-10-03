# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**PokerHole** is an event-sourced Texas Hold'em poker system with a hexagonal architecture. The server is Java/Spring Boot, the client is Go with a terminal UI.

**Core Principle**: Event Sourcing + Server Authority + Offline-First

---

## Essential Commands

### Server (Java/Spring Boot)

```bash
cd pokerhole-server

# Start PostgreSQL
docker compose up -d

# Run server (port 8080)
./gradlew bootRun

# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests HandEvaluatorTest

# Run architecture tests
./gradlew test --tests "*ArchitectureTest"

# Run golden vector tests (21 cases, Java-Go parity)
./gradlew test --tests "*GoldenVectorValidationTest*"

# Generate coverage report
./gradlew test jacocoTestReport
open build/reports/jacoco/test/html/index.html

# Build JAR
./gradlew build

# Debug mode (port 5005)
./gradlew bootRun --debug-jvm
```

### Client (Go)

```bash
cd pokerhole-cli

# Run client
go run cmd/poker-client/main.go

# Run all tests
go test ./...

# Run golden tests (verify Java-Go parity)
go test -v ./tests/golden/

# Run specific test
go test -v ./internal/domain/card

# Coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Build executable
go build -o poker cmd/poker-client/main.go

# Production build (optimized)
go build -ldflags="-s -w" -o poker cmd/poker-client/main.go
```

### Verification Commands

```bash
# Count server tests (should be 502)
find pokerhole-server -name "*.java" | xargs grep "@Test" | wc -l

# List golden vectors (should be 21)
cat pokerhole-server/src/test/resources/golden/hand_eval.json | grep -c '"case_id"'

# Check project structure
find . -name "*.md" -type f | grep -v ".archive" | grep -v ".git"
```

---

## Architecture Deep Dive

### Hexagonal Architecture (Ports & Adapters)

**Critical Rule**: Dependencies point INWARD. Domain has ZERO framework dependencies.

```
[Input Adapters]  →  [Application]  →  [Domain]  →  [Output Adapters]
WebSocket/REST      Use Cases         Pure Logic    DB/Network/Event
```

**Server Package Structure**:
- `core/domain/`: Pure Java, no Spring, enforced by ArchUnit
  - `game/`: Game aggregate (Texas Hold'em rules, round orchestration)
  - `player/`: Player aggregate (chips, actions, status)
  - `card/`: Card, Deck, deterministic shuffling (seeded RNG)
  - `service/`: HandEvaluator, PotManager (domain services)
  - `shared/`: DomainEvent, AggregateRoot interfaces

- `core/application/`: Use cases, port interfaces
  - `port/in/`: StartGameUseCase, PlaceBetUseCase (driving ports)
  - `port/out/`: GameRepositoryPort, EventStorePort (driven ports)
  - `service/`: Use case implementations

- `adapter/in/`: Input adapters (driving)
  - `websocket/`: GameWebSocketHandler, message types, session registry
  - `web/`: REST controllers (future)

- `adapter/out/`: Output adapters (driven)
  - `persistence/`: JPA entities, repositories, adapters
  - `event/`: EventStoreAdapter (PostgreSQL append-only log)
  - `matching/`: In-memory matching queue
  - `ai/`: RuleBasedAIStrategy

**Client Package Structure** (Go):
- `internal/domain/`: Pure Go port of Java domain (golden-tested for parity)
  - `game/`: Game, Round (same logic as Java)
  - `player/`: Player aggregate
  - `card/`: Card, Deck (same deterministic shuffle)
  - `evaluator/`: HandEvaluator (21 golden test cases)

- `internal/application/`: Use cases, ports (mirrors server)
  - `port/in/`, `port/out/`: Same interfaces as server

- `internal/adapter/`:
  - `ui/`: Bubble Tea TUI (splash, menu, game screens)
  - `storage/`: SQLite event store
  - `network/`: WebSocket client (auto-reconnect, heartbeat)
  - `crypto/`: ed25519 event signing

### Event Sourcing Architecture

**Every state change is an event**. Current state = replay events from beginning (or from snapshot).

**Event Flow** (see ADR-001, ADR-002):

1. **Client submits**: Signed event proposal with `client_seq`
2. **Server validates**: Domain rules + signature verification
3. **Server assigns**: `server_seq` (authoritative order)
4. **Server appends**: To PostgreSQL event log (immutable)
5. **Server broadcasts**: To all connected players

**Domain Events** (past tense, immutable):
- `GameStarted`: Game initialized with players, seed
- `PlayerActed`: CALL/RAISE/FOLD/CHECK/ALL_IN
- `RoundProgressed`: PRE_FLOP → FLOP → TURN → RIVER
- `RoundEnded`: Winners determined, pots distributed
- `GameEnded`: Game finished

**Event Store Schema** (PostgreSQL):
```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    game_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    server_seq BIGSERIAL,       -- Authoritative order
    applied_at TIMESTAMP NOT NULL,
    validator_version VARCHAR(20)
);
```

**Client Event Store** (SQLite):
```sql
CREATE TABLE events (
    event_id TEXT PRIMARY KEY,
    game_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    payload TEXT NOT NULL,      -- JSON
    timestamp INTEGER NOT NULL,
    signature BLOB,              -- ed25519
    synced BOOLEAN DEFAULT 0    -- Offline→online sync flag
);
```

### Domain-Driven Design Patterns

**Aggregates** (consistency boundaries):
- `Game`: Root entity for game state
  - Methods: `startGame()`, `playerAction()`, `progressRound()`, `endRound()`
  - Emits domain events via `pullDomainEvents()`
  - Enforces Texas Hold'em rules

- `Player`: Root entity for player state
  - Manages chips, actions, status (ACTIVE/FOLDED/ALL_IN)

**Value Objects** (immutable):
- `Card(suit, rank)`: Immutable card
- `Deck`: 52 cards, deterministic shuffle with seed (ADR-005)
- `Pot`, `SidePot`: Money management
- `HandResult(tier, kickers)`: Poker hand ranking
- `PlayerId`, `GameId`: Type-safe IDs

**Domain Services** (stateless logic):
- `HandEvaluator`: Evaluates 5-card hands → HandResult (Tier + kickers)
- `PotManager`: Distributes pots (main pot + side pots for all-ins)

### Golden Test Vectors (Java ↔ Go Parity)

**21 test cases** ensure identical behavior:

**Server**: `pokerhole-server/src/test/resources/golden/hand_eval.json`
**Client**: `pokerhole-cli/tests/golden/data/hand_eval.json` (same file)

**Test Coverage**:
- Royal Flush (2 cases)
- Straight Flush including A-2-3-4-5 wheel (2 cases)
- Four of a Kind (3 cases), Full House (3 cases), Flush (2 cases)
- Straight (3 cases), Three of a Kind, Two Pair, One Pair, High Card (6 cases)

**Run Tests**:
```bash
# Server
./gradlew test --tests "*GoldenVectorValidationTest*"

# Client
go test -v ./tests/golden/
```

**Critical**: If you change hand evaluation logic in Java, you MUST update Go and verify golden tests pass on both sides.

---

## Key Architecture Decisions (ADRs)

Read these FIRST before making changes:

### ADR-001: Event Sourcing
- **Why**: Audit trail, replay capability, offline→online sync
- **Trade-off**: More storage (mitigated by snapshots), complex queries (use projections)
- **Impact**: All state changes must be events, never update state directly

### ADR-002: Server Authority
- **Why**: Prevent cheating, ensure fairness
- **Flow**: Client proposes → Server validates → Server orders → Server broadcasts
- **Impact**: Server is single source of truth, clients are "dumb terminals" for online play

### ADR-003: ed25519 Signatures
- **Why**: Tamper detection, offline→online sync verification
- **How**: Client signs all events with private key, server verifies
- **Impact**: Every event must include valid signature

### ADR-004: Snapshot Strategy (planned)
- **Why**: Performance (avoid replaying millions of events)
- **How**: Snapshot every N events
- **Impact**: Replay = snapshot + events since snapshot

### ADR-005: Deterministic RNG
- **Why**: Fairness verification, same shuffle on Java and Go
- **How**: Seeded shuffling, golden tests verify parity
- **Impact**: `Deck.shuffle(seed)` must produce identical results on Java and Go

---

## WebSocket Protocol

### Connection
```
ws://localhost:8080/ws/game
```

### Message Format (JSON)
```json
{
  "type": "MESSAGE_TYPE",
  "timestamp": 1234567890,
  "payload": { /* type-specific */ }
}
```

### Client → Server Messages
- `REGISTER`: `{uuid, nickname}` - Initial connection
- `HEARTBEAT`: `{}` - Keep-alive (every 30s)
- `JOIN_RANDOM_MATCH`: `{}` - Join random matching
- `CALL`, `RAISE`, `FOLD`, `CHECK`, `ALL_IN`: Game actions

### Server → Client Messages
- `REGISTER_SUCCESS`: `{playerId}` - Registration confirmed
- `GAME_STATE_UPDATE`: `{full game state}` - State sync
- `PLAYER_ACTION`: `{playerId, action}` - Action notification
- `MATCHING_COMPLETED`: `{gameId, players}` - Game starting
- `ERROR`: `{message}` - Error occurred

**Critical**: Server authority means client actions are PROPOSALS. Server validates and may reject.

---

## Development Patterns

### Adding a New Domain Event

1. **Define event** (Java):
```java
public record PlayerTimedOut(
    UUID eventId,
    UUID aggregateId,
    Instant occurredAt,
    PlayerId playerId
) implements DomainEvent {}
```

2. **Port to Go**:
```go
type PlayerTimedOut struct {
    EventID    string
    GameID     string
    OccurredAt time.Time
    PlayerID   PlayerID
}
```

3. **Update aggregate** to emit event
4. **Add event handler** in application layer
5. **Update WebSocket protocol** if needed
6. **Add tests** for both Java and Go

### Adding a New Use Case

1. **Define port interface** (`core/application/port/in/`):
```java
public interface TimeoutPlayerUseCase {
    void execute(TimeoutPlayerCommand command);
}
```

2. **Implement service** (`core/application/service/`):
```java
@Service
@RequiredArgsConstructor
public class TimeoutPlayerService implements TimeoutPlayerUseCase {
    private final GameRepositoryPort gameRepository;
    private final EventStorePort eventStore;

    @Override
    public void execute(TimeoutPlayerCommand command) {
        // Load game, apply timeout, save events
    }
}
```

3. **Create adapter** (e.g., WebSocket handler or REST controller)
4. **Wire in config** if needed
5. **Add tests**: unit (domain), integration (adapter)

### Testing Strategy

**Server Tests (502 total)**:
- **Domain logic** (~400): Pure Java, no mocks, 100% coverage required
- **Architecture** (~4): ArchUnit enforces hexagonal rules
- **Golden vectors** (~8): Hand evaluation parity with Go
- **JPA** (~25): Repository integration tests
- **WebSocket** (~20): Protocol tests
- **Other** (~45): Matching, AI, event store

**Client Tests**:
- **Golden tests**: MUST pass to ensure Java-Go parity
- **Domain logic**: Mirror Java test coverage
- **TUI**: Bubble Tea component tests
- **Integration**: WebSocket, SQLite event store

### Code Review Checklist

Before submitting changes:

**Server**:
- [ ] Domain has zero Spring dependencies (ArchUnit enforces)
- [ ] All domain logic has tests (coverage ≥ 95%)
- [ ] Golden tests pass (if hand evaluation changed)
- [ ] No breaking changes to WebSocket protocol
- [ ] Events are immutable (use records or `@Value`)
- [ ] Use `Optional<T>` instead of null returns

**Client**:
- [ ] Golden tests pass (Java-Go parity verified)
- [ ] Domain logic mirrors Java implementation
- [ ] Events signed with ed25519
- [ ] Error handling uses `fmt.Errorf("context: %w", err)`
- [ ] Imports grouped: stdlib → external → internal

---

## Common Pitfalls

### ❌ Don't: Modify domain state directly
```java
// BAD
game.setRound(BettingRound.FLOP);
```

### ✅ Do: Emit events and apply
```java
// GOOD
List<DomainEvent> events = game.progressRound();
eventStore.append(events);
```

### ❌ Don't: Add Spring annotations to domain
```java
// BAD
@Service // This is in core/domain/
public class HandEvaluator { }
```

### ✅ Do: Keep domain pure
```java
// GOOD (core/domain/service/)
public class HandEvaluator {
    public HandResult evaluate(List<Card> cards) { }
}
```

### ❌ Don't: Trust client state
```java
// BAD
game.applyAction(clientMessage.getAction()); // No validation
```

### ✅ Do: Validate everything
```java
// GOOD
if (!game.isValidAction(playerId, action)) {
    return ErrorMessage.ILLEGAL_ACTION;
}
game.playerAction(playerId, action, amount);
```

### ❌ Don't: Break Java-Go parity
```java
// BAD - changing shuffle without updating Go
public void shuffle(Random rng) {
    Collections.shuffle(cards, rng); // Different algorithm
}
```

### ✅ Do: Maintain deterministic parity
```java
// GOOD - same Fisher-Yates as Go
public void shuffle(long seed) {
    Random rng = new Random(seed);
    // Fisher-Yates implementation
}
```

---

## Troubleshooting

### Server won't start
```bash
# Check PostgreSQL
docker compose ps
docker compose logs postgres

# Check port conflict
lsof -i :8080
```

### Tests failing after dependency update
```bash
./gradlew clean build --refresh-dependencies
```

### Golden tests failing
```bash
# Sync golden data from server to client
cp pokerhole-server/src/test/resources/golden/hand_eval.json \
   pokerhole-cli/tests/golden/data/

# Re-run both
./gradlew test --tests "*GoldenVectorValidationTest*"
go test -v ./tests/golden/
```

### WebSocket connection refused
```bash
# Verify server health
curl http://localhost:8080/actuator/health

# Test WebSocket endpoint
wscat -c ws://localhost:8080/ws/game
```

### Client database locked (SQLite)
```bash
rm ~/.pokerhole/poker.db
# Restart client
```

---

## Project Status

**Last Verified**: 2025-10-03

- **Server**: Phase 1 ~75% complete (502 tests passing)
- **Client**: Phase 2 complete (Go-Java parity verified via 21 golden tests)
- **Integration**: Phase 3 in progress (online multiplayer)

**Documentation Structure** (9 files total):
- `./README.md`: Project overview
- `./docs/adr/*.md`: Architecture Decision Records (read these!)
- `./pokerhole-server/README.md`: Server details
- `./pokerhole-cli/README.md`: Client details

**Philosophy**: ADR tells WHY, code tells HOW, README tells WHAT.
