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
# Count server tests (currently 49 - Phase 1 Steps 1-7 complete)
./gradlew test --no-daemon
# Breakdown: 18 TexasHoldemIntegration + 25 HandEvaluator + 4 Architecture + 2 other

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
- `START_TEXAS_HOLDEM`: `{}` - Start game (room host only)
- `CALL`, `RAISE`, `FOLD`, `CHECK`, `ALL_IN`: Game actions

### Server → Client Messages
- `REGISTER_SUCCESS`: `{playerId}` - Registration confirmed
- `GAME_STARTED`: `{gameId, players}` - Texas Hold'em started
- `GAME_STATE_UPDATE`: `{full game state}` - State sync
- `PLAYER_ACTION`: `{playerId, action}` - Action notification
- `TURN_CHANGED`: `{currentPlayer}` - Turn changed (Phase 1 Step 7)
- `ROUND_PROGRESSED`: `{round, cards}` - Round advanced (Phase 1 Step 7)
- `MATCHING_COMPLETED`: `{gameId, players}` - Game starting
- `ERROR`: `{message}` - Error occurred

### Protocol Changes (Phase 1 Step 7)

**Server-client synchronization completed**. See `/PROTOCOL-COMPARISON.md` for full details.

**Critical field changes**:
1. `roomId` → `gameId` (clearer naming for game context)
2. `currentTurnPlayer` → `currentPlayer` (simpler)
3. Player object: `currentBet` → `bet` (consistency)
4. Community cards: `[{"suit":"HEARTS","rank":"ACE"}]` → `["AH"]` (70% payload reduction)

**New message types added**:
- `TURN_CHANGED` (server notifies turn changes)
- `ROUND_PROGRESSED` (server notifies round advancement with new cards)
- `CHAT_MESSAGE` (future feature, structure prepared)

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

**Server Tests (49 currently, Phase 1 Steps 1-7)**:
- **Texas Hold'em Integration** (18): Full game flow, all actions, round progression, error cases
- **Hand Evaluator** (25): Poker hand ranking (21 golden vectors for Java-Go parity)
- **Architecture** (4): ArchUnit enforces hexagonal rules
- **JPA** (1): Repository integration tests
- **Application** (1): Spring Boot context loading
- **Target for Phase 2**: ~500+ tests (domain logic, WebSocket protocol, event store, matching, AI)

**Critical Bug Fixes Found via Testing** (Phase 1 Step 6):
- **Bug #1**: Betting round completion - Fixed to check if all ACTIVE players have acted (not just bet amounts)
- **Bug #2**: CHECK action recording - Fixed to record CHECK in `currentRoundBets` map

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

**Last Verified**: 2025-10-04

- **Server**: Phase 1 87.5% complete (49 tests passing, Steps 1-7 done, Step 8 in progress)
  - ✅ Texas Hold'em game logic fully implemented (300+ lines in Dealer.java)
  - ✅ Player actions (FOLD, CHECK, CALL, RAISE, ALL_IN) working
  - ✅ Betting round progression (PRE_FLOP → FLOP → TURN → RIVER → SHOWDOWN)
  - ✅ Winner determination via HandEvaluator (21 golden tests)
  - ✅ WebSocket real-time communication
  - ✅ Server-client protocol synchronized (4 field changes, 3 message types added)
  - ✅ 2 critical bugs fixed (betting round logic, CHECK action recording)
  - ⚠️ Known limitations: side pots, blinds, timeouts (Phase 2)

- **Client**: Phase 2 structure complete (0 tests, golden tests pending)
- **Integration**: Server-client protocol aligned ✅ (see /PROTOCOL-COMPARISON.md)

**Documentation Structure** (10 files total):
- `./README.md`: Project overview
- `./NEXT-STEP.md`: Current progress and next steps (Phase 1 Step 8)
- `./PROTOCOL-COMPARISON.md`: Server-client protocol comparison (Step 7)
- `./docs/adr/*.md`: Architecture Decision Records (read these!)
- `./pokerhole-server/README.md`: Server details with Texas Hold'em gameplay
- `./pokerhole-cli/README.md`: Client details

**Philosophy**: ADR tells WHY, code tells HOW, README tells WHAT.
- 이 프로젝트는 서버, ㅍ크랄르 관리하는 통합 프로젝트다 그래서 작업을 하려면 각 디렉토리로 들어가서 작업을 하고 다시 올라와야한다. cd ./pokerhole-server , cd ./pokerhole-cli  모든 문서는 지정하지 않으면 루트에 생성하고
- # 개발자 필수 사전 지식

**프로젝트**: PokerHole (Event-sourced Texas Hold'em)
**작성일**: 2025-10-03
**대상**: 이 프로젝트를 처음 시작하는 개발자

---

## 목적

이 프로젝트는 **여러 고급 아키텍처 패턴과 도메인 지식**을 결합한 복잡한 시스템입니다.
새로운 개발자가 효과적으로 기여하기 위해 아래 개념들을 이해하는 것을 권장합니다.

---

## 아키텍처 패턴 (필수)

### 1. Hexagonal Architecture (육각형 아키텍처, Ports & Adapters)

**핵심 개념**:
- 도메인 로직을 외부 기술(DB, 웹, UI)로부터 완전히 분리
- 의존성은 항상 **내부(도메인)**를 향함
- 포트(Port) = 인터페이스, 어댑터(Adapter) = 구현체

**구조**:
```
[Input Adapters]  →  [Application Layer]  →  [Domain Layer]  ←  [Output Adapters]
  WebSocket/REST      Use Cases               Pure Business        DB/Event Store
```

**왜 중요한가**:
- 도메인 로직 테스트가 쉬움 (프레임워크 의존성 없음)
- 기술 스택 교체 용이 (예: PostgreSQL → MongoDB)
- ArchUnit 테스트로 규칙 강제

**학습 자료**:
- https://alistair.cockburn.us/hexagonal-architecture/
- "Get Your Hands Dirty on Clean Architecture" (책)

---

### 2. Event Sourcing (이벤트 소싱)

**핵심 개념**:
- 상태를 직접 저장하지 않고, **상태 변경 이벤트**를 저장
- 현재 상태 = 처음부터 모든 이벤트를 재생(replay)한 결과
- 이벤트는 **불변(immutable)**, 과거는 절대 바뀌지 않음

**예시**:
```
상태 기반 (기존):
  User(id=1, name="Alice", balance=1000)

이벤트 기반 (Event Sourcing):
  1. UserCreated(id=1, name="Alice")
  2. BalanceDeposited(id=1, amount=500)
  3. BalanceDeposited(id=1, amount=500)
  → 현재 balance = 0 + 500 + 500 = 1000
```

**장점**:
- 완전한 감사 추적(audit trail)
- 시간 여행 가능 (과거 시점 상태 재현)
- 버그 디버깅 용이
- 오프라인→온라인 동기화 (이 프로젝트의 핵심!)

**단점**:
- 저장 공간 증가 (→ 스냅샷으로 완화)
- 쿼리 복잡도 증가 (→ 프로젝션/뷰로 완화)

**학습 자료**:
- https://martinfowler.com/eaaDev/EventSourcing.html
- "Event Sourcing" by Greg Young (YouTube)

---

### 3. Domain-Driven Design (DDD)

**핵심 개념**:
- 복잡한 비즈니스 로직을 **도메인 모델**로 표현
- 유비쿼터스 언어(Ubiquitous Language): 개발자와 도메인 전문가가 같은 용어 사용
- 전략적 설계: Bounded Context, Aggregate
- 전술적 설계: Entity, Value Object, Domain Service

**이 프로젝트의 DDD 요소**:
- **Aggregate Root**: `Game`, `Player`
- **Value Object**: `Card`, `Deck`, `PlayerId`, `GameId`
- **Domain Event**: `GameStarted`, `PlayerActed`, `RoundProgressed`
- **Domain Service**: `HandEvaluator`, `PotManager`

**왜 중요한가**:
- 복잡한 Texas Hold'em 규칙을 체계적으로 모델링
- 비즈니스 로직과 기술 로직 명확히 분리

**학습 자료**:
- "Domain-Driven Design" by Eric Evans (책, 필독!)
- "Implementing Domain-Driven Design" by Vaughn Vernon

---

### 4. CQRS (Command Query Responsibility Segregation)

**핵심 개념**:
- 명령(Command): 상태 변경 (쓰기)
- 쿼리(Query): 상태 조회 (읽기)
- 둘을 **완전히 분리**

**이 프로젝트의 CQRS**:
- **Command**: `StartGameUseCase`, `PlaceBetUseCase` (도메인 이벤트 생성)
- **Query**: 이벤트 스토어에서 이벤트를 읽어 상태 재구성

**왜 Event Sourcing과 잘 맞는가**:
- 이벤트 저장 = Command 측
- 이벤트 읽기 + 재생 = Query 측
- 자연스럽게 분리됨

**학습 자료**:
- https://martinfowler.com/bliki/CQRS.html
- https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs

---

## 도메인 지식 (필수)

### 5. Texas Hold'em 포커 규칙

**핵심 개념**:
- 각 플레이어에게 **홀카드 2장** (비공개)
- **커뮤니티 카드 5장** (공개, 모든 플레이어 공유)
- 4번의 베팅 라운드: PRE_FLOP → FLOP → TURN → RIVER → SHOWDOWN
- 최종 패는 홀카드 2장 + 커뮤니티 5장 중 **최고의 5장 조합**

**베팅 라운드**:
1. **PRE_FLOP**: 홀카드 2장 받은 후
2. **FLOP**: 커뮤니티 3장 공개 후
3. **TURN**: 커뮤니티 1장 추가 (총 4장) 후
4. **RIVER**: 커뮤니티 1장 추가 (총 5장) 후
5. **SHOWDOWN**: 남은 플레이어들이 패 공개, 승자 결정

**플레이어 액션**:
- **FOLD**: 게임 포기
- **CHECK**: 베팅 없이 패스 (현재 베팅이 0일 때만)
- **CALL**: 현재 베팅에 맞춤
- **RAISE**: 현재 베팅보다 높게 베팅
- **ALL_IN**: 모든 칩을 베팅

**패 순위** (높은 순):
1. Royal Flush: A-K-Q-J-10 (같은 무늬)
2. Straight Flush: 연속된 5장 (같은 무늬)
3. Four of a Kind: 4장이 같은 숫자
4. Full House: Three of a Kind + One Pair
5. Flush: 5장이 같은 무늬
6. Straight: 연속된 5장
7. Three of a Kind: 3장이 같은 숫자
8. Two Pair: 2개의 페어
9. One Pair: 1개의 페어
10. High Card: 위의 조합이 없을 때, 가장 높은 카드

**왜 중요한가**:
- 이 프로젝트의 **핵심 비즈니스 로직**
- 규칙을 모르면 코드 이해 불가

**학습 자료**:
- https://www.pokernews.com/poker-rules/texas-holdem.htm
- YouTube: "How to Play Texas Hold'em Poker"

---

### 6. 사이드 팟 (Side Pot)

**핵심 개념**:
- 플레이어가 ALL_IN했을 때, 그 이상의 베팅은 **별도의 팟**으로 분리
- ALL_IN 플레이어는 메인 팟만 가져갈 수 있음

**예시**:
```
Player A: 1,000 칩 ALL_IN
Player B: 2,000 칩 베팅
Player C: 2,000 칩 베팅

결과:
- Main Pot: 3,000 (A, B, C가 경쟁)
- Side Pot: 2,000 (B, C만 경쟁)

승자 결정:
- A가 최고 패: A는 Main Pot 3,000만 가져감, Side Pot 2,000은 B/C 중 승자
- B가 최고 패: B는 Main Pot 3,000 + Side Pot 2,000 = 5,000
```

**왜 중요한가**:
- Texas Hold'em의 복잡한 부분
- 여러 사이드 팟이 생길 수 있음 (3명 이상 ALL_IN)

**학습 자료**:
- https://www.pokernews.com/poker-rules/side-pots.htm

---

## 보안 및 암호화 (선택)

### 7. ed25519 디지털 서명

**핵심 개념**:
- 공개키 암호화 알고리즘 (Public Key Cryptography)
- 개인키로 서명 → 공개키로 검증
- 타원곡선 암호화 기반 (빠르고 안전)

**이 프로젝트에서의 사용**:
- 클라이언트가 모든 이벤트에 서명
- 서버가 서명 검증 → 위변조 방지
- 오프라인 게임 → 온라인 동기화 시 신뢰성 확보

**왜 중요한가**:
- 부정행위 방지
- 오프라인 이벤트의 무결성 보장

**학습 자료**:
- https://ed25519.cr.yp.to/
- "Practical Cryptography for Developers" (책)

---

### 8. Deterministic RNG (결정론적 난수 생성)

**핵심 개념**:
- 같은 시드(seed)를 주면 항상 같은 난수 시퀀스 생성
- 게임 공정성 검증에 필수

**이 프로젝트에서의 사용**:
```java
// 시드 12345로 덱 섞기
Deck deck = Deck.newDeck();
deck.shuffle(12345L);

// Java와 Go에서 같은 시드 → 같은 순서의 카드
```

**왜 중요한가**:
- 클라이언트와 서버가 **같은 덱 순서**를 보장
- 게임 재생(replay) 가능
- 공정성 검증 가능

**학습 자료**:
- ADR-005: Deterministic RNG Fairness
- Fisher-Yates Shuffle Algorithm

---

## 통신 프로토콜 (필수)

### 9. WebSocket 실시간 통신

**핵심 개념**:
- HTTP와 달리, **양방향 실시간 통신**
- 연결 유지 (persistent connection)
- 서버 → 클라이언트 푸시 가능

**이 프로젝트에서의 사용**:
```
Client                    Server
  |                         |
  |--- REGISTER ----------->|
  |<-- REGISTER_SUCCESS ----|
  |                         |
  |--- JOIN_RANDOM_MATCH -->|
  |<-- MATCHING_COMPLETED --|
  |                         |
  |--- CALL ---------------->|
  |<-- PLAYER_ACTION --------|
  |<-- TURN_CHANGED ---------|
```

**왜 HTTP가 아닌 WebSocket인가**:
- 게임은 **실시간 상호작용** 필요
- HTTP는 요청-응답 모델 (서버가 먼저 보낼 수 없음)
- WebSocket은 서버가 능동적으로 푸시 가능

**학습 자료**:
- https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- Spring WebSocket Documentation

---

### 10. Server Authority (서버 권한 모델)

**핵심 개념**:
- 클라이언트는 **제안(proposal)**만 보냄
- 서버가 **최종 검증 및 결정**
- 부정행위 방지

**이 프로젝트에서의 예시**:
```
잘못된 방식 (Client Authority):
  Client: "나 RAISE 1,000,000!" (클라이언트가 칩 부족한데도 전송)
  Server: "OK!" (검증 없이 수락) → 부정행위 가능

올바른 방식 (Server Authority):
  Client: "RAISE 1,000,000 제안"
  Server: "칩 부족, 거부!" → 서버가 검증
```

**왜 중요한가**:
- 온라인 게임의 기본 원칙
- 클라이언트는 해킹될 수 있음 (Memory Editor, Cheat Engine)
- 서버만 신뢰할 수 있음

**학습 자료**:
- ADR-002: Server Authority
- "Multiplayer Game Programming" by Joshua Glazer (책)

---

## 테스트 전략 (필수)

### 11. Golden Test Vectors (언어 간 Parity)

**핵심 개념**:
- 같은 입력 → 같은 출력을 **여러 언어**에서 보장
- JSON 파일로 테스트 케이스 정의
- Java와 Go가 같은 JSON 파일 사용

**이 프로젝트에서의 사용**:
```
hand_eval.json:
{
  "case_id": "royal_flush_01",
  "cards": ["AS", "KS", "QS", "JS", "TS"],
  "expected_tier": "ROYAL_FLUSH",
  "expected_kickers": []
}

Java 테스트:
  HandEvaluator.evaluate(cards) == ROYAL_FLUSH

Go 테스트:
  handEvaluator.Evaluate(cards) == ROYAL_FLUSH
```

**왜 중요한가**:
- Java 서버와 Go 클라이언트가 **같은 로직** 보장
- 한쪽을 수정하면 양쪽 테스트 실행 필수

**학습 자료**:
- 프로젝트의 `tests/golden/` 디렉토리
- "Table-Driven Tests" in Go

---

### 12. 아키텍처 테스트 (ArchUnit)

**핵심 개념**:
- 코드로 아키텍처 규칙 강제
- 예: "도메인 레이어는 Spring 의존성 가질 수 없음"

**이 프로젝트에서의 사용**:
```java
@Test
void domainLayerShouldNotDependOnSpring() {
    noClasses()
        .that().resideInPackage("..core.domain..")
        .should().dependOnClassesThat()
        .resideInPackage("org.springframework..")
        .check(importedClasses);
}
```

**왜 중요한가**:
- Hexagonal Architecture 규칙 자동 검증
- 실수로 의존성 추가하는 것 방지

**학습 자료**:
- https://www.archunit.org/
- "ArchUnit User Guide"

---

## 데이터베이스 및 저장소 (선택)

### 13. Append-Only 로그 (불변 저장소)

**핵심 개념**:
- 데이터를 **절대 수정하지 않음**, 추가만 함
- Event Sourcing과 완벽히 일치

**이 프로젝트에서의 사용**:
```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    game_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    server_seq BIGSERIAL,       -- 항상 증가
    applied_at TIMESTAMP NOT NULL
);

-- UPDATE 없음, INSERT만 있음
INSERT INTO events (event_id, game_id, event_type, payload, applied_at)
VALUES ('uuid', 'game-123', 'PlayerActed', '{"action": "CALL"}', NOW());
```

**왜 중요한가**:
- 감사 추적 보장
- 데이터 무결성
- 롤백 불필요 (과거 시점 재생하면 됨)

---

### 14. PostgreSQL JSONB

**핵심 개념**:
- JSON 데이터를 **바이너리 형태**로 저장
- 인덱싱 및 쿼리 가능

**이 프로젝트에서의 사용**:
```sql
-- 특정 플레이어의 액션만 조회
SELECT * FROM events
WHERE event_type = 'PlayerActed'
  AND payload->>'playerId' = 'uuid-123';

-- JSONB 인덱스
CREATE INDEX idx_events_payload_player ON events
USING GIN ((payload->'playerId'));
```

**왜 중요한가**:
- 이벤트 페이로드를 유연하게 저장
- 스키마 변경 없이 새로운 필드 추가 가능

**학습 자료**:
- PostgreSQL JSONB Documentation
- "PostgreSQL: Up and Running" (책)

---

## 프론트엔드 (Go 클라이언트, 선택)

### 15. Bubble Tea (Go TUI 프레임워크)

**핵심 개념**:
- The Elm Architecture (Model-Update-View) 패턴
- 터미널 UI를 React처럼 작성

**구조**:
```go
type Model struct {
    view ViewMode
    gameState GameState
}

func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // 메시지 처리 → 상태 업데이트
}

func (m Model) View() string {
    // 상태 → UI 렌더링
}
```

**왜 중요한가**:
- 클라이언트 구현에 사용
- 반응형 UI 패턴 이해 필요

**학습 자료**:
- https://github.com/charmbracelet/bubbletea
- Elm Architecture: https://guide.elm-lang.org/architecture/

---

## 개발 도구 및 기술

### 16. Gradle (Java 빌드 도구)

**핵심 개념**:
- Java 프로젝트 빌드 자동화
- 의존성 관리

**이 프로젝트에서의 사용**:
```bash
./gradlew build        # 빌드
./gradlew test         # 테스트
./gradlew bootRun      # 서버 실행
```

---

### 17. Go Modules (Go 의존성 관리)

**핵심 개념**:
- Go 1.11+의 공식 의존성 관리
- `go.mod` 파일로 버전 관리

**이 프로젝트에서의 사용**:
```bash
go mod download        # 의존성 다운로드
go mod tidy            # 사용하지 않는 의존성 제거
go test ./...          # 모든 테스트 실행
```

---

### 18. Docker Compose

**핵심 개념**:
- 여러 컨테이너를 정의하고 실행
- 개발 환경 구성 자동화

**이 프로젝트에서의 사용**:
```bash
cd pokerhole-server
docker compose up -d   # PostgreSQL 실행
```

---

## 우선순위 학습 가이드

### 필수 (1-2주 집중 학습)

1. **Hexagonal Architecture** - 프로젝트 전체 구조 이해
2. **Event Sourcing** - 핵심 데이터 모델
3. **Texas Hold'em 규칙** - 비즈니스 로직
4. **Domain-Driven Design** - 도메인 모델링
5. **WebSocket** - 통신 프로토콜
6. **Server Authority** - 보안 모델

### 권장 (기여 시작 후 학습 가능)

7. **사이드 팟** - 복잡한 비즈니스 로직
8. **CQRS** - 읽기/쓰기 분리
9. **ed25519** - 보안 검증
10. **Deterministic RNG** - 재현 가능한 난수

### 선택 (필요 시 학습)

11. Golden Test Vectors
12. ArchUnit
13. PostgreSQL JSONB
14. Bubble Tea (클라이언트 작업 시)

---

## 학습 로드맵 (4주 추천)

### Week 1: 아키텍처 이해
- [ ] Hexagonal Architecture 개념
- [ ] Event Sourcing 개념
- [ ] 프로젝트 README 정독
- [ ] ADR (Architecture Decision Records) 읽기

### Week 2: 도메인 지식
- [ ] Texas Hold'em 규칙 숙지
- [ ] 실제 포커 게임 플레이 (온라인)
- [ ] DDD 기본 개념
- [ ] 프로젝트 도메인 모델 분석

### Week 3: 기술 스택
- [ ] Spring Boot + WebSocket
- [ ] PostgreSQL + JSONB
- [ ] Gradle 빌드 시스템
- [ ] 기존 테스트 실행 및 분석

### Week 4: 실전 기여
- [ ] 간단한 이슈 해결 (Good First Issue)
- [ ] 테스트 작성
- [ ] 코드 리뷰 참여
- [ ] 문서 기여

---

## 추천 도서

1. **"Domain-Driven Design"** by Eric Evans - DDD 바이블
2. **"Implementing Domain-Driven Design"** by Vaughn Vernon - 실전 DDD
3. **"Patterns, Principles, and Practices of Domain-Driven Design"** by Scott Millett
4. **"Building Event-Driven Microservices"** by Adam Bellemare - Event Sourcing
5. **"Get Your Hands Dirty on Clean Architecture"** by Tom Hombergs - Hexagonal Architecture

---

## 온라인 강의

1. **Hexagonal Architecture**: https://www.youtube.com/watch?v=th4AgBcrEHA
2. **Event Sourcing**: https://www.youtube.com/watch?v=8JKjvY4etTY (Greg Young)
3. **DDD**: https://www.pluralsight.com/courses/domain-driven-design-fundamentals
4. **Texas Hold'em**: https://www.youtube.com/watch?v=GAoR9ji8D6A

---

## FAQ

### Q1: 모든 개념을 다 알아야 기여할 수 있나요?
A: 아니요! **필수** 카테고리만 이해하면 시작할 수 있습니다.

### Q2: 포커를 해본 적이 없는데 괜찮나요?
A: 괜찮습니다만, **Texas Hold'em 규칙**은 반드시 학습해야 합니다. 온라인 포커 시뮬레이터로 연습하세요.

### Q3: Java는 알지만 Go는 모르는데요?
A: 서버(Java)에만 기여하면 됩니다. 클라이언트(Go)는 별도 작업입니다.

### Q4: 이 모든 걸 배우는 데 얼마나 걸리나요?
A: **필수 개념** 기준으로 2-3주, **전체** 기준으로 2-3개월 예상합니다.

---

## 도움 받기

- **GitHub Issues**: 질문 올리기
- **CLAUDE.md**: 프로젝트 가이드 읽기
- **ADR 문서**: 아키텍처 결정 이유 확인
- **기존 테스트**: 코드 예시로 활용

---

**Good luck!**
- # Texas Hold'em 턴제 시스템 설계

**작성일**: 2025-10-03
**목적**: 서버를 단순 5장 비교 게임 → Texas Hold'em 턴제 게임으로 확장

---

## Texas Hold'em 기본 규칙

### 게임 흐름

```
1. PRE_FLOP:  각 플레이어에게 홀카드 2장 배분 → 베팅 라운드
2. FLOP:      커뮤니티 카드 3장 공개 → 베팅 라운드
3. TURN:      커뮤니티 카드 1장 추가 (총 4장) → 베팅 라운드
4. RIVER:     커뮤니티 카드 1장 추가 (총 5장) → 베팅 라운드
5. SHOWDOWN:  남은 플레이어들이 패 공개 → 승자 결정
```

### 베팅 라운드 규칙

**액션 타입**:
- **FOLD**: 게임 포기
- **CHECK**: 베팅 없이 패스 (현재 베팅이 0일 때만)
- **CALL**: 현재 베팅에 맞춤
- **RAISE**: 현재 베팅보다 높게 베팅
- **ALL_IN**: 모든 칩을 베팅

**베팅 라운드 종료 조건**:
- 모든 플레이어가 동일한 금액을 베팅했거나
- 한 명을 제외한 모든 플레이어가 FOLD했거나
- 남은 플레이어가 모두 ALL_IN 상태

---

## 아키텍처 설계

### Phase 1: 도메인 모델 확장

#### 1.1 새로운 Enum 타입

```java
// BettingRound.java
public enum BettingRound {
    PRE_FLOP,   // 홀카드 2장 배분 후
    FLOP,       // 커뮤니티 3장 공개 후
    TURN,       // 커뮤니티 4장 공개 후
    RIVER,      // 커뮤니티 5장 공개 후
    SHOWDOWN    // 최종 패 공개
}

// PlayerAction.java
public enum PlayerAction {
    FOLD,       // 게임 포기
    CHECK,      // 베팅 없이 패스
    CALL,       // 현재 베팅에 맞춤
    RAISE,      // 현재 베팅보다 높게
    ALL_IN      // 모든 칩 베팅
}

// PlayerStatus.java
public enum PlayerStatus {
    WAITING,    // 대기 중
    ACTIVE,     // 게임 중
    FOLDED,     // 폴드함
    ALL_IN,     // 올인함
    OUT         // 칩이 없어 탈락
}
```

#### 1.2 도메인 이벤트 추가

```java
// PlayerActed.java
public record PlayerActed(
    UUID eventId,
    UUID gameId,
    Instant occurredAt,
    PlayerId playerId,
    PlayerAction action,
    int amount,             // RAISE의 경우 금액
    BettingRound round      // 어느 라운드에서 발생했는지
) implements DomainEvent {}

// RoundProgressed.java
public record RoundProgressed(
    UUID eventId,
    UUID gameId,
    Instant occurredAt,
    BettingRound fromRound,
    BettingRound toRound,
    List<Card> newCommunityCards
) implements DomainEvent {}

// TurnChanged.java
public record TurnChanged(
    UUID eventId,
    UUID gameId,
    Instant occurredAt,
    PlayerId currentPlayerId,
    int timeoutSeconds      // 타임아웃 시간 (예: 30초)
) implements DomainEvent {}

// PotDistributed.java
public record PotDistributed(
    UUID eventId,
    UUID gameId,
    Instant occurredAt,
    Map<PlayerId, Integer> winners,  // 승자와 받은 금액
    int totalPot
) implements DomainEvent {}
```

### Phase 2: Dealer 확장

#### 2.1 새로운 상태 관리

```java
public class Dealer {
    // 기존 필드
    private Deck deck;
    private final List<Player> players;

    // 새로운 필드 (Texas Hold'em)
    private BettingRound currentRound;
    private List<Card> communityCards;      // 커뮤니티 카드
    private int currentBet;                 // 현재 베팅 금액
    private int pot;                        // 팟
    private List<SidePot> sidePots;         // 사이드 팟 (ALL_IN 처리)
    private int dealerButtonPosition;       // 딜러 버튼 위치
    private int currentPlayerPosition;      // 현재 턴 플레이어

    // 베팅 라운드 추적
    private Map<PlayerId, Integer> currentRoundBets;  // 이번 라운드 베팅액
    private int lastRaisePosition;          // 마지막 레이즈한 플레이어 위치
}
```

#### 2.2 핵심 메서드

```java
// 게임 시작 (홀카드 2장 배분)
public void startTexasHoldem() {
    this.deck = Deck.newDeck();
    this.deck.shuffle();
    this.communityCards = new ArrayList<>();
    this.currentRound = BettingRound.PRE_FLOP;
    this.currentBet = 0;
    this.pot = 0;
    this.sidePots = new ArrayList<>();
    this.currentRoundBets = new HashMap<>();

    // 각 플레이어에게 2장씩 배분
    for (int i = 0; i < 2; i++) {
        for (Player player : players) {
            Card card = deck.drawCard();
            player.receiveCard(card);
        }
    }

    // 첫 번째 플레이어부터 시작 (실제로는 딜러 버튼 다음)
    this.currentPlayerPosition = 0;
}

// 플레이어 액션 처리
public void processPlayerAction(PlayerId playerId, PlayerAction action, int amount) {
    Player player = findPlayer(playerId);

    // 액션 유효성 검증
    validateAction(player, action, amount);

    switch (action) {
        case FOLD:
            player.setStatus(PlayerStatus.FOLDED);
            break;

        case CHECK:
            // 아무 것도 하지 않음 (베팅이 0일 때만 가능)
            if (currentBet > 0) {
                throw new IllegalStateException("현재 베팅이 있을 때는 CHECK할 수 없습니다.");
            }
            break;

        case CALL:
            int callAmount = currentBet - currentRoundBets.getOrDefault(playerId, 0);
            player.bet(callAmount);
            pot += callAmount;
            currentRoundBets.put(playerId, currentBet);
            break;

        case RAISE:
            int raiseAmount = amount;
            if (raiseAmount <= currentBet) {
                throw new IllegalStateException("RAISE는 현재 베팅보다 높아야 합니다.");
            }
            int toBet = raiseAmount - currentRoundBets.getOrDefault(playerId, 0);
            player.bet(toBet);
            pot += toBet;
            currentBet = raiseAmount;
            currentRoundBets.put(playerId, raiseAmount);
            lastRaisePosition = currentPlayerPosition;
            break;

        case ALL_IN:
            int allInAmount = player.getChips();
            player.bet(allInAmount);
            pot += allInAmount;
            player.setStatus(PlayerStatus.ALL_IN);

            // ALL_IN 금액이 현재 베팅보다 높으면 레이즈 효과
            int totalBet = currentRoundBets.getOrDefault(playerId, 0) + allInAmount;
            if (totalBet > currentBet) {
                currentBet = totalBet;
                lastRaisePosition = currentPlayerPosition;
            }
            currentRoundBets.put(playerId, totalBet);
            break;
    }

    // 다음 플레이어로 턴 이동
    moveToNextPlayer();

    // 베팅 라운드 종료 확인
    if (isBettingRoundComplete()) {
        progressToNextRound();
    }
}

// 베팅 라운드 종료 확인
private boolean isBettingRoundComplete() {
    // 한 명만 남은 경우
    long activePlayers = players.stream()
        .filter(p -> p.getStatus() == PlayerStatus.ACTIVE ||
                     p.getStatus() == PlayerStatus.ALL_IN)
        .count();
    if (activePlayers <= 1) {
        return true;
    }

    // 모든 플레이어가 동일한 금액을 베팅했는지 확인
    long playersWhoNeedToAct = players.stream()
        .filter(p -> p.getStatus() == PlayerStatus.ACTIVE)
        .filter(p -> {
            int playerBet = currentRoundBets.getOrDefault(p.getId(), 0);
            return playerBet < currentBet;
        })
        .count();

    return playersWhoNeedToAct == 0;
}

// 다음 라운드로 진행
private void progressToNextRound() {
    // 현재 라운드 베팅 초기화
    currentRoundBets.clear();
    currentBet = 0;

    switch (currentRound) {
        case PRE_FLOP:
            // FLOP: 커뮤니티 3장 공개
            communityCards.add(deck.drawCard());
            communityCards.add(deck.drawCard());
            communityCards.add(deck.drawCard());
            currentRound = BettingRound.FLOP;
            break;

        case FLOP:
            // TURN: 커뮤니티 1장 추가
            communityCards.add(deck.drawCard());
            currentRound = BettingRound.TURN;
            break;

        case TURN:
            // RIVER: 커뮤니티 1장 추가
            communityCards.add(deck.drawCard());
            currentRound = BettingRound.RIVER;
            break;

        case RIVER:
            // SHOWDOWN: 패 공개 및 승자 결정
            currentRound = BettingRound.SHOWDOWN;
            determineWinner();
            break;

        case SHOWDOWN:
            // 게임 종료
            break;
    }

    // 첫 번째 ACTIVE 플레이어부터 다시 시작
    currentPlayerPosition = findNextActivePlayer(dealerButtonPosition);
}

// 승자 결정
private void determineWinner() {
    // 각 플레이어의 최종 핸드 평가 (홀카드 2장 + 커뮤니티 5장)
    List<Player> activePlayers = players.stream()
        .filter(p -> p.getStatus() != PlayerStatus.FOLDED)
        .collect(Collectors.toList());

    // HandEvaluator로 각 플레이어의 패 평가
    HandEvaluator evaluator = new HandEvaluator();
    Map<Player, HandResult> results = new HashMap<>();

    for (Player player : activePlayers) {
        List<Card> allCards = new ArrayList<>();
        allCards.addAll(player.getHand());  // 홀카드 2장
        allCards.addAll(communityCards);     // 커뮤니티 5장

        // 7장 중 최고의 5장 조합 찾기
        HandResult best = evaluator.findBestHand(allCards);
        results.put(player, best);
    }

    // 승자 결정 및 팟 분배
    distributePot(results);
}

// 팟 분배 (사이드 팟 처리 포함)
private void distributePot(Map<Player, HandResult> results) {
    // TODO: 사이드 팟 로직 구현
    // 메인 팟과 사이드 팟을 각각 처리

    // 단순화: 최고 패를 가진 플레이어에게 전체 팟 지급
    Player winner = results.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElseThrow();

    winner.addChips(pot);
    pot = 0;
}
```

### Phase 3: GameRoom 통합

#### 3.1 GameRoom 메서드 추가

```java
public class GameRoom {
    // 기존 필드
    private final Dealer dealer;
    private final Map<String, Participant> participants;

    // 새로운 메서드: 게임 액션 처리
    public void processPlayerAction(SessionState session, PlayerAction action, int amount) {
        synchronized (this) {
            Participant participant = participants.get(session.id());
            if (participant == null) {
                throw new IllegalStateException("방에 참가하지 않았습니다.");
            }

            Player player = participant.player();
            PlayerId playerId = player.getId();

            // Dealer에 액션 위임
            dealer.processPlayerAction(playerId, action, amount);

            // 게임 상태 브로드캐스트
            broadcastGameState();
        }
    }

    // 게임 상태 브로드캐스트 (WebSocket)
    private void broadcastGameState() {
        // TODO: GameCommandService와 연동
        // 모든 참가자에게 현재 게임 상태 전송
    }

    // 현재 턴 플레이어 가져오기
    public Optional<Player> getCurrentTurnPlayer() {
        return dealer.getCurrentTurnPlayer();
    }
}
```

### Phase 4: GameCommandService 구현

#### 4.1 executeAction 메서드 구현

```java
@Service
@RequiredArgsConstructor
public class GameCommandService {

    private final RoomRegistry roomRegistry;
    private final WebSocketSessionRegistry sessionRegistry;
    private final ObjectMapper objectMapper;  // JSON 직렬화

    public void executeAction(String sessionId, String action, Integer amount) {
        PlayerSession playerSession = sessionRegistry.findBySessionId(sessionId)
                .orElseThrow(() -> new IllegalStateException("등록되지 않은 세션입니다."));

        String roomId = playerSession.getCurrentRoomId();
        if (roomId == null) {
            throw new IllegalStateException("방에 참가하지 않았습니다.");
        }

        GameRoom room = roomRegistry.findRoom(roomId)
                .orElseThrow(() -> new IllegalStateException("방을 찾을 수 없습니다: " + roomId));

        // 액션 변환
        PlayerAction playerAction = PlayerAction.valueOf(action);

        try {
            // GameRoom에 액션 처리 위임
            room.processPlayerAction(playerSession.getSession(), playerAction, amount != null ? amount : 0);

            // 성공 시 모든 참가자에게 브로드캐스트
            broadcastPlayerAction(room, playerSession, playerAction, amount);

        } catch (IllegalStateException | IllegalArgumentException e) {
            // 실패 시 에러 메시지 전송
            sendError(playerSession.getSession(), e.getMessage());
        }
    }

    private void broadcastPlayerAction(GameRoom room, PlayerSession playerSession,
                                       PlayerAction action, Integer amount) {
        Map<String, Object> actionPayload = Map.of(
                "playerId", playerSession.getUuid(),
                "nickname", playerSession.getNickname(),
                "action", action.name(),
                "amount", amount != null ? amount : 0
        );

        ServerMessage message = ServerMessage.of(ServerMessageType.PLAYER_ACTION, actionPayload);

        // 방의 모든 참가자에게 전송
        broadcastToRoom(room, message);
    }

    // GameRoom의 participants에 접근하기 위한 헬퍼
    private void broadcastToRoom(GameRoom room, ServerMessage message) {
        // Option 1: GameRoom에 broadcastWebSocket 메서드 추가
        // Option 2: RoomRegistry에서 참가자 목록 관리
        // TODO: 구현 필요
    }
}
```

### Phase 5: WebSocket 프로토콜 확장

#### 5.1 새로운 메시지 타입

```java
// ClientMessageType.java
public enum ClientMessageType {
    REGISTER,
    HEARTBEAT,
    JOIN_RANDOM_MATCH,

    // 게임 액션 (새로 추가)
    CALL,
    RAISE,
    FOLD,
    CHECK,
    ALL_IN
}

// ServerMessageType.java
public enum ServerMessageType {
    REGISTER_SUCCESS,
    MATCHING_COMPLETED,
    GAME_STATE_UPDATE,

    // 게임 이벤트 (새로 추가)
    PLAYER_ACTION,          // 플레이어 액션 통보
    TURN_CHANGED,           // 턴 변경 통보
    ROUND_PROGRESSED,       // 라운드 진행 (FLOP/TURN/RIVER)
    POT_DISTRIBUTED,        // 팟 분배 결과
    ERROR
}
```

#### 5.2 메시지 페이로드

```json
// PLAYER_ACTION (Server → Client)
{
  "type": "PLAYER_ACTION",
  "timestamp": 1234567890,
  "payload": {
    "playerId": "uuid-123",
    "nickname": "Player1",
    "action": "RAISE",
    "amount": 200,
    "round": "FLOP"
  }
}

// TURN_CHANGED (Server → Client)
{
  "type": "TURN_CHANGED",
  "timestamp": 1234567890,
  "payload": {
    "currentPlayerId": "uuid-456",
    "currentPlayerNickname": "Player2",
    "timeoutSeconds": 30
  }
}

// ROUND_PROGRESSED (Server → Client)
{
  "type": "ROUND_PROGRESSED",
  "timestamp": 1234567890,
  "payload": {
    "fromRound": "PRE_FLOP",
    "toRound": "FLOP",
    "communityCards": [
      {"suit": "HEARTS", "rank": "ACE"},
      {"suit": "DIAMONDS", "rank": "KING"},
      {"suit": "CLUBS", "rank": "QUEEN"}
    ]
  }
}

// GAME_STATE_UPDATE (Server → Client, 전체 상태 동기화)
{
  "type": "GAME_STATE_UPDATE",
  "timestamp": 1234567890,
  "payload": {
    "gameId": "game-uuid",
    "round": "FLOP",
    "pot": 1000,
    "currentBet": 200,
    "communityCards": [
      {"suit": "HEARTS", "rank": "ACE"},
      {"suit": "DIAMONDS", "rank": "KING"},
      {"suit": "CLUBS", "rank": "QUEEN"}
    ],
    "players": [
      {
        "playerId": "uuid-123",
        "nickname": "Player1",
        "chips": 8000,
        "status": "ACTIVE",
        "currentBet": 200,
        "hand": [  // 자신의 홀카드만 볼 수 있음
          {"suit": "SPADES", "rank": "JACK"},
          {"suit": "HEARTS", "rank": "TEN"}
        ]
      },
      {
        "playerId": "uuid-456",
        "nickname": "Player2",
        "chips": 9500,
        "status": "ACTIVE",
        "currentBet": 200,
        "hand": []  // 다른 플레이어의 카드는 숨김
      }
    ],
    "currentTurnPlayerId": "uuid-456"
  }
}
```

---

## 구현 순서

### Step 1: 도메인 모델 확장 (1-2일)
1. Enum 타입 생성 (BettingRound, PlayerAction, PlayerStatus)
2. 도메인 이벤트 추가 (PlayerActed, RoundProgressed, TurnChanged, PotDistributed)
3. Player에 상태 관리 추가 (chips, status)
4. 단위 테스트 작성

**예상 파일**:
- `core/domain/game/BettingRound.java`
- `core/domain/game/PlayerAction.java`
- `core/domain/player/PlayerStatus.java`
- `core/domain/shared/PlayerActed.java`
- `core/domain/shared/RoundProgressed.java`
- `core/domain/shared/TurnChanged.java`
- `core/domain/shared/PotDistributed.java`

### Step 2: Dealer 확장 (3-4일)
1. 새로운 필드 추가 (currentRound, communityCards, pot, etc.)
2. startTexasHoldem() 메서드 구현
3. processPlayerAction() 메서드 구현
4. isBettingRoundComplete() 로직 구현
5. progressToNextRound() 로직 구현
6. determineWinner() 로직 구현 (HandEvaluator 활용)
7. distributePot() 로직 구현 (사이드 팟 처리)
8. 단위 테스트 작성 (모든 액션, 모든 라운드)

**예상 테스트**:
- `DealerTexasHoldemTest.java` (시나리오 기반)
- 각 액션별 테스트 (FOLD, CHECK, CALL, RAISE, ALL_IN)
- 각 라운드 전환 테스트
- 승자 결정 테스트
- 사이드 팟 테스트

### Step 3: GameRoom 통합 (1일)
1. processPlayerAction() 메서드 추가
2. broadcastGameState() 메서드 추가
3. getCurrentTurnPlayer() 메서드 추가
4. startRound()를 Texas Hold'em 방식으로 수정
5. 통합 테스트 작성

### Step 4: GameCommandService 구현 (1-2일)
1. executeAction() TODO 구현
2. broadcastPlayerAction() 구현
3. broadcastToRoom() 구현 (participants 접근 문제 해결)
4. GameRoom에 WebSocket 브로드캐스트 메서드 추가
5. 통합 테스트 작성

**해결 방안**:
- **Option A**: GameRoom에 `broadcastWebSocket(ServerMessage)` 메서드 추가
- **Option B**: RoomRegistry에서 참가자 WebSocket 세션 관리

### Step 5: WebSocket 프로토콜 확장 (1일)
1. ClientMessageType에 CALL, RAISE, FOLD, CHECK, ALL_IN 추가
2. ServerMessageType에 PLAYER_ACTION, TURN_CHANGED, ROUND_PROGRESSED, POT_DISTRIBUTED 추가
3. GameWebSocketHandler에서 새로운 메시지 타입 처리
4. 페이로드 DTO 생성
5. WebSocket 통합 테스트 작성

### Step 6: 통합 테스트 (2-3일)
1. 전체 게임 플로우 E2E 테스트
2. 여러 시나리오 테스트:
   - 2명 게임 (기본)
   - 4명 게임 (최대)
   - 중간에 플레이어 FOLD
   - ALL_IN 및 사이드 팟
   - 타임아웃 처리
3. WebSocket 메시지 플로우 검증
4. 클라이언트 연동 테스트

### Step 7: 클라이언트 동기화 (1일)
1. 클라이언트 Go 코드 확인
2. 프로토콜 일치 여부 검증
3. 필요 시 클라이언트 수정
4. 서버-클라이언트 통합 테스트

### Step 8: 문서 업데이트 (1일)
1. ADR-002 수정 (Server Authority + Texas Hold'em)
2. README.md 업데이트
3. NEXT-STEP.md 업데이트
4. pokerhole-server/README.md 업데이트

---

## 주의사항

### 아키텍처 제약
1. **Hexagonal Architecture 준수**:
   - 도메인 레이어는 Spring 의존성 없음
   - ArchUnit 테스트 통과 필수

2. **Event Sourcing**:
   - 모든 상태 변경은 이벤트로 기록
   - 이벤트는 immutable (record 사용)

3. **Server Authority**:
   - 클라이언트 액션은 제안(proposal)
   - 서버가 최종 검증 및 승인

### 기술적 고려사항

#### 사이드 팟 (Side Pot)
- ALL_IN 플레이어가 있을 때 복잡함
- 예:
  - Player A: 1000 칩 ALL_IN
  - Player B: 2000 칩 베팅
  - Player C: 2000 칩 베팅
  - Main Pot: 3000 (A, B, C)
  - Side Pot: 2000 (B, C only)

#### 턴 타임아웃
- 30초 내에 액션 없으면 자동 FOLD
- Timer 관리 필요 (Spring @Scheduled or ScheduledExecutorService)

#### 동시성 처리
- GameRoom의 synchronized 블록 유지
- 여러 플레이어의 동시 액션 방지

---

## 테스트 전략

### 단위 테스트
- Dealer의 각 메서드 단위
- 모든 액션 타입 (FOLD, CHECK, CALL, RAISE, ALL_IN)
- 모든 라운드 전환 (PRE_FLOP → FLOP → TURN → RIVER → SHOWDOWN)
- 승자 결정 로직
- 사이드 팟 로직

### 통합 테스트
- GameRoom + Dealer 통합
- WebSocket 메시지 플로우
- 여러 플레이어 시나리오

### E2E 테스트
- 서버 실행 → 클라이언트 연결 → 게임 플레이 → 승자 결정
- 서버-클라이언트 프로토콜 검증

---

## 예상 일정

| 단계 | 작업 내용 | 예상 기간 | 우선순위 |
|------|-----------|-----------|----------|
| Step 1 | 도메인 모델 확장 | 1-2일 | P0 |
| Step 2 | Dealer 확장 | 3-4일 | P0 |
| Step 3 | GameRoom 통합 | 1일 | P0 |
| Step 4 | GameCommandService 구현 | 1-2일 | P0 |
| Step 5 | WebSocket 프로토콜 확장 | 1일 | P0 |
| Step 6 | 통합 테스트 | 2-3일 | P1 |
| Step 7 | 클라이언트 동기화 | 1일 | P1 |
| Step 8 | 문서 업데이트 | 1일 | P2 |

**총 예상 기간**: 10-14일 (2-3주)

---

## 성공 기준

1. 서버가 Texas Hold'em 규칙대로 동작
2. 모든 플레이어 액션 처리 가능 (FOLD, CHECK, CALL, RAISE, ALL_IN)
3. 베팅 라운드 자동 전환 (PRE_FLOP → FLOP → TURN → RIVER → SHOWDOWN)
4. 승자 결정 및 팟 분배 정확
5. 사이드 팟 처리 정확
6. WebSocket 프로토콜 일치
7. 클라이언트와 통합 테스트 성공
8. 기존 테스트 31개 모두 통과 (회귀 방지)
9. 새로운 테스트 추가 (목표: +50 tests)

---

## 참고 자료

- **Texas Hold'em Rules**: https://www.pokernews.com/poker-rules/texas-holdem.htm
- **ADR-001**: Event Sourcing for Gameplay
- **ADR-002**: Server Authority (업데이트 필요)
- **ADR-005**: Deterministic RNG Fairness
- **HandEvaluator**: 기존 패 평가 로직 재사용

---

**다음 단계**: Step 1 시작 - 도메인 모델 확장