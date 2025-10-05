# CLAUDE.md - LLM Work Instructions

This file provides work rules and conventions for Claude Code when working with this codebase.

---

## Project Overview

**PokerHole** is an event-sourced Texas Hold'em poker system with hexagonal architecture.

**Core Principles**:
- Event Sourcing: All state changes are immutable events
- Hexagonal Architecture: Domain has ZERO framework dependencies
- Server Authority: Client proposes, server validates
- Offline-First: Client can work offline, sync later

**Tech Stack**:
- Server: Java 23 + Spring Boot + PostgreSQL
- Client: Go + Terminal UI (Bubble Tea)
- Communication: WebSocket (real-time)

---

## Essential Commands

### Server (Java/Spring Boot)
```bash
cd pokerhole-server

# Start PostgreSQL
docker compose up -d

# Run server (port 8080)
./gradlew bootRun

# Run tests
./gradlew test

# Run specific test
./gradlew test --tests "ClassName"

# Architecture tests
./gradlew test --tests "*ArchitectureTest"
```

### Client (Go)
```bash
cd pokerhole-cli

# Run client
go run cmd/poker-client/main.go

# Run tests
go test ./...

# Golden tests (Java-Go parity)
go test -v ./tests/golden/
```

---

## Architecture Rules

### 1. Hexagonal Architecture (CRITICAL)

**Dependency Rule**: Dependencies point INWARD only.

```
[Input Adapters] → [Application] → [Domain] ← [Output Adapters]
WebSocket/REST     Use Cases       Pure Logic    DB/Event Store
```

**Package Structure**:
- `core/domain/`: Pure Java, NO Spring dependencies (enforced by ArchUnit)
- `core/application/`: Use cases, port interfaces
- `adapter/in/`: WebSocket handlers, REST controllers
- `adapter/out/`: JPA repositories, event store

**Rules**:
- Domain layer MUST NOT import Spring classes
- Domain layer MUST NOT import adapter classes
- Application layer MAY import domain
- Adapters MAY import application and domain

**ArchUnit enforces these rules - tests will FAIL if violated.**

---

### 2. Event Sourcing Pattern

**Every state change is an event.**

**Event Flow**:
1. Client submits signed event proposal
2. Server validates (domain rules + signature)
3. Server assigns server_seq (authoritative order)
4. Server appends to PostgreSQL event log (immutable)
5. Server broadcasts to all connected players

**Domain Events** (past tense, immutable):
- `GameStarted`: Game initialized
- `PlayerActed`: CALL/RAISE/FOLD/CHECK/ALL_IN
- `RoundProgressed`: PRE_FLOP → FLOP → TURN → RIVER
- `RoundEnded`: Winners determined, pots distributed

**Critical Rules**:
- Events are IMMUTABLE (use Java `record`)
- NEVER update existing events
- Current state = replay all events from beginning
- Use `Optional<T>` instead of null returns

---

### 3. Domain-Driven Design Patterns

**Aggregates** (consistency boundaries):
- `Game`: Root entity, enforces Texas Hold'em rules
- `Player`: Root entity, manages chips/actions/status

**Value Objects** (immutable):
- `Card`, `Deck`, `Pot`, `SidePot`, `HandResult`
- `PlayerId`, `GameId` (type-safe IDs)

**Domain Services** (stateless):
- `HandEvaluator`: Evaluates poker hands
- `PotManager`: Distributes pots (main + side pots)

---

## Dependency Rules

### Domain Layer (`core/domain/`)

**ALLOWED**:
- Java standard library (java.util, java.time)
- Domain interfaces and classes

**FORBIDDEN**:
- `org.springframework.*`
- `jakarta.persistence.*`
- Any adapter/infrastructure classes

**Example**:
```java
// GOOD - Pure domain
public class HandEvaluator {
    public HandResult evaluate(List<Card> cards) { }
}

// BAD - Spring annotation in domain
@Service  // FORBIDDEN
public class HandEvaluator { }
```

---

## Code Conventions

### Java

**1. Use `record` for immutable data**:
```java
// Events
public record PlayerActed(
    UUID eventId,
    UUID gameId,
    Instant occurredAt,
    PlayerId playerId,
    PlayerAction action
) implements DomainEvent {}

// Value Objects
public record Card(Suit suit, Rank rank) {}
```

**2. Return `Optional<T>` instead of null**:
```java
// GOOD
public Optional<Player> findPlayer(PlayerId id) {
    return players.stream()
        .filter(p -> p.getId().equals(id))
        .findFirst();
}

// BAD
public Player findPlayer(PlayerId id) {
    return null;  // AVOID
}
```

**3. Package imports**:
- Group imports: java stdlib → external → internal
- No wildcard imports

### Go

**1. Error handling**:
```go
// GOOD
if err != nil {
    return fmt.Errorf("context: %w", err)
}

// BAD
if err != nil {
    return err  // Lost context
}
```

**2. Import grouping**:
```go
import (
    // stdlib
    "fmt"
    "time"

    // external
    "github.com/charmbracelet/bubbletea"

    // internal
    "github.com/xiyo/pokerhole-cli/internal/domain"
)
```

---

## Game Rules (Texas Hold'em)

### Betting Rounds
1. **PRE_FLOP**: After receiving 2 hole cards
2. **FLOP**: After 3 community cards dealt
3. **TURN**: After 4th community card
4. **RIVER**: After 5th community card
5. **SHOWDOWN**: Reveal hands, determine winner

### Player Actions
- **FOLD**: Quit the hand
- **CHECK**: Pass (only if currentBet == 0)
- **CALL**: Match current bet
- **RAISE**: Bet higher than current
- **ALL_IN**: Bet all chips

### Hand Rankings (High to Low)
1. Royal Flush: A-K-Q-J-10 (same suit)
2. Straight Flush: 5 consecutive cards (same suit)
3. Four of a Kind: 4 cards same rank
4. Full House: 3 of a kind + pair
5. Flush: 5 cards same suit
6. Straight: 5 consecutive cards
7. Three of a Kind: 3 cards same rank
8. Two Pair: 2 pairs
9. One Pair: 1 pair
10. High Card: Highest card wins

### Betting Round Completion
Round ends when:
- All ACTIVE players have acted AND same bet amount, OR
- Only 1 ACTIVE player remains (others FOLDED), OR
- All remaining players are ALL_IN

---

## WebSocket Protocol

### Connection
```
ws://localhost:8080/ws/game
```

### Message Format
```json
{
  "type": "MESSAGE_TYPE",
  "timestamp": 1234567890,
  "payload": { }
}
```

### Client → Server
- `REGISTER`: Initial connection
- `HEARTBEAT`: Keep-alive (every 30s)
- `JOIN_RANDOM_MATCH`: Join matching queue
- `START_TEXAS_HOLDEM`: Start game (host only)
- `CALL`, `RAISE`, `FOLD`, `CHECK`, `ALL_IN`: Game actions

### Server → Client
- `REGISTER_SUCCESS`: Registration confirmed
- `GAME_STARTED`: Game initialized
- `GAME_STATE_UPDATE`: Full state sync
- `PLAYER_ACTION`: Action notification
- `TURN_CHANGED`: Turn changed
- `ROUND_PROGRESSED`: Round advanced
- `ERROR`: Error occurred

**Server Authority**: Client actions are PROPOSALS. Server validates and may REJECT.

---

## Development Patterns

### Adding a Domain Event

**1. Define event (Java)**:
```java
public record PlayerTimedOut(
    UUID eventId,
    UUID gameId,
    Instant occurredAt,
    PlayerId playerId
) implements DomainEvent {}
```

**2. Port to Go**:
```go
type PlayerTimedOut struct {
    EventID    string
    GameID     string
    OccurredAt time.Time
    PlayerID   PlayerID
}
```

**3. Update aggregate to emit event**
**4. Add event handler in application layer**
**5. Update WebSocket protocol if needed**
**6. Add tests for both Java and Go**

---

### Adding a Use Case

**1. Define port interface** (`core/application/port/in/`):
```java
public interface TimeoutPlayerUseCase {
    void execute(TimeoutPlayerCommand command);
}
```

**2. Implement service** (`core/application/service/`):
```java
@Service
@RequiredArgsConstructor
public class TimeoutPlayerService implements TimeoutPlayerUseCase {
    private final GameRepositoryPort gameRepository;
    private final EventStorePort eventStore;

    @Override
    public void execute(TimeoutPlayerCommand command) {
        // Implementation
    }
}
```

**3. Create adapter** (WebSocket handler or REST controller)
**4. Add tests**: unit (domain), integration (adapter)

---

## Code Change Rules

### Rule 1: Events are IMMUTABLE
```java
// BAD - Modifying state directly
game.setRound(BettingRound.FLOP);

// GOOD - Emit events
List<DomainEvent> events = game.progressRound();
eventStore.append(events);
```

### Rule 2: No Spring in Domain
```java
// BAD - Spring in domain
@Service  // core/domain/ package
public class HandEvaluator { }

// GOOD - Pure domain
public class HandEvaluator {  // core/domain/ package
    public HandResult evaluate(List<Card> cards) { }
}
```

### Rule 3: Server Validates Everything
```java
// BAD - Trust client
game.applyAction(clientMessage.getAction());

// GOOD - Validate first
if (!game.isValidAction(playerId, action)) {
    return ErrorMessage.ILLEGAL_ACTION;
}
game.playerAction(playerId, action, amount);
```

### Rule 4: Maintain Java-Go Parity
**Golden tests verify identical behavior.**

If you change hand evaluation in Java:
1. Update Go implementation
2. Run golden tests on BOTH sides:
   ```bash
   ./gradlew test --tests "*GoldenVectorValidationTest*"
   go test -v ./tests/golden/
   ```

---

## Code Review Checklist

**Before Commit**:
- [ ] Domain has zero Spring dependencies (ArchUnit passes)
- [ ] All domain logic has tests (coverage ≥ 95%)
- [ ] Golden tests pass (if hand evaluation changed)
- [ ] No breaking changes to WebSocket protocol
- [ ] Events are immutable (use `record`)
- [ ] Use `Optional<T>` instead of null returns
- [ ] All tests passing

---

## Common Pitfalls

### Don't: Modify domain state directly
```java
game.setRound(BettingRound.FLOP);  // BAD
```

### Do: Emit events and apply
```java
List<DomainEvent> events = game.progressRound();  // GOOD
eventStore.append(events);
```

### Don't: Add Spring annotations to domain
```java
@Service  // BAD - in core/domain/
public class HandEvaluator { }
```

### Do: Keep domain pure
```java
public class HandEvaluator {  // GOOD - in core/domain/service/
    public HandResult evaluate(List<Card> cards) { }
}
```

### Don't: Trust client state
```java
game.applyAction(clientMessage.getAction());  // BAD - no validation
```

### Do: Validate everything
```java
if (!game.isValidAction(playerId, action)) {  // GOOD
    return ErrorMessage.ILLEGAL_ACTION;
}
game.playerAction(playerId, action, amount);
```

### Don't: Break Java-Go parity
```java
// BAD - changing shuffle without updating Go
public void shuffle(Random rng) {
    Collections.shuffle(cards, rng);  // Different algorithm
}
```

### Do: Maintain deterministic parity
```java
// GOOD - same Fisher-Yates as Go
public void shuffle(long seed) {
    Random rng = new Random(seed);
    // Fisher-Yates implementation
}
```

---

## Work Workflow

### BEFORE Starting Work
**MANDATORY**: Read `ROADMAP.md`
- Check current project status
- Review next steps
- Understand known limitations
- Check if your task conflicts with planned work

### DURING Work
**Follow**:
1. Architecture rules (Hexagonal, Event Sourcing, DDD)
2. Dependency rules (no Spring in domain)
3. Code conventions (Java record, Go error handling)
4. Code change rules (events immutable, validate everything)

### AFTER Completing Work
**MANDATORY**:
1. Update `ROADMAP.md` with:
   - Current status changes
   - Completed items marked
   - New known limitations (if any)
   - Updated test counts
2. Update `workslog.md` (see below)
3. Run all tests (`./gradlew test`)
4. Verify no regressions

---

## workslog.md Management

### IMPORTANT: Append-Only Pattern

**NEVER use Read or Edit tools on workslog.md**

**CORRECT - Use Bash append**:
```bash
cat << 'EOF' | tee -a workslog.md

## 2025-10-04 13:30 - Task Title

### Summary
Work summary...

### Changes
- Change 1
- Change 2

---

EOF
```

**WRONG**:
- `Read` tool (file too large, slow)
- `Edit` tool (must read entire file, inefficient)

**Rules**:
- workslog.md is append-only
- New entries go to END of file
- User will organize/cleanup manually

---

## Project Directory Structure

This is a **parent project managing server (Java) and client (Go)**.

**Enter subdirectories to work**:
- `cd pokerhole-server` (Java/Spring Boot server)
- `cd pokerhole-cli` (Go client)

**Documents default to root** unless specified otherwise.

---

## Golden Test Vectors (Java ↔ Go Parity)

**21 test cases** ensure identical behavior across languages.

**Files**:
- Server: `pokerhole-server/src/test/resources/golden/hand_eval.json`
- Client: `pokerhole-cli/tests/golden/data/hand_eval.json`

**Test Coverage**:
- Royal Flush (2 cases)
- Straight Flush (2 cases)
- Four of a Kind (3 cases)
- Full House (3 cases)
- Flush (2 cases)
- Straight (3 cases)
- Three of a Kind, Two Pair, One Pair, High Card (6 cases)

**Run Tests**:
```bash
# Server
./gradlew test --tests "*GoldenVectorValidationTest*"

# Client
go test -v ./tests/golden/
```

**CRITICAL**: If you change hand evaluation logic in Java, you MUST:
1. Update Go implementation identically
2. Verify golden tests pass on BOTH sides
3. If test cases change, update JSON file and sync to both sides

---

## Key Architecture Decisions (ADRs)

Read these before making changes:

**ADR-001: Event Sourcing**
- Why: Audit trail, replay capability, offline→online sync
- Impact: All state changes must be events

**ADR-002: Server Authority**
- Why: Prevent cheating, ensure fairness
- Impact: Server is single source of truth

**ADR-003: ed25519 Signatures**
- Why: Tamper detection, offline sync verification
- Impact: Every event must include valid signature

**ADR-005: Deterministic RNG**
- Why: Fairness verification, same shuffle on Java and Go
- Impact: `Deck.shuffle(seed)` produces identical results across languages

Full ADR documents: `docs/adr/*.md`

---

## Testing Strategy

**Server** (151 tests total):
- HandEvaluatorTest: 25 (Golden vectors)
- TexasHoldemIntegrationTest: 18
- BlindsTest: 7
- SidePotTest: 9
- TurnTimeoutTest: 7
- HandResultTest: 34
- PotDistributorTest: 24
- WinnerResolverTest: 14
- SplitPotIntegrationTest: 7
- ArchitectureTest: 4
- Others: 2

**Client**:
- Golden tests MUST pass (Java-Go parity)
- Domain logic mirrors Java coverage
- TUI: Bubble Tea component tests
- Integration: WebSocket, SQLite event store

---

## Troubleshooting

### Server won't start
```bash
docker compose ps
docker compose logs postgres
lsof -i :8080
```

### Tests failing
```bash
./gradlew clean build --refresh-dependencies
```

### Golden tests failing
```bash
# Sync from server to client
cp pokerhole-server/src/test/resources/golden/hand_eval.json \
   pokerhole-cli/tests/golden/data/

# Re-run both
./gradlew test --tests "*GoldenVectorValidationTest*"
go test -v ./tests/golden/
```

### WebSocket connection refused
```bash
curl http://localhost:8080/actuator/health
wscat -c ws://localhost:8080/ws/game
```

---

## Critical Reminders

1. **ALWAYS read ROADMAP.md BEFORE starting work**
2. **ALWAYS update ROADMAP.md AFTER completing work**
3. **NEVER add Spring dependencies to core/domain/**
4. **NEVER modify events (immutable)**
5. **ALWAYS validate client input (Server Authority)**
6. **ALWAYS maintain Java-Go parity (golden tests)**
7. **ALWAYS run all tests before commit**
8. **NEVER use Read/Edit on workslog.md (append-only with Bash)**

---

**Work Flow Summary**:
```
1. Read ROADMAP.md
2. Follow CLAUDE.md rules
3. Implement with tests
4. Update ROADMAP.md
5. Append to workslog.md (Bash)
6. Run all tests
7. Commit
```
- 모든 문제를 풀때까지 재귀적으로 계속해서 동작해라 제일 중요하다!!!!!
- 이모지 절때 사용 금지
- 항상 디자인은 버블티, 버블스, 립슬로스를 꼭써서디자인해라 디자인이 불가느한건반드시 주인님한테 물어봐라
- 이모지 절때 금지, ㅇ어디서든!!! 절대금지