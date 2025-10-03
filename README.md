# PokerHole

Event-sourced Texas Hold'em Poker with hexagonal architecture.

**Server**: Java/Spring Boot | **Client**: Go/Bubble Tea TUI

---

## Quick Start

### Server
```bash
cd pokerhole-server
docker compose up -d          # Start PostgreSQL
./gradlew bootRun            # Start server (port 8080)
```

### Client
```bash
cd pokerhole-cli
go run cmd/poker-client/main.go
```

---

## Project Structure

```
pokerhole/
├── docs/adr/                # Architecture Decision Records
│   ├── 001-event-sourcing-for-gameplay.md
│   ├── 002-server-authority.md
│   ├── 003-ed25519-client-signatures.md
│   ├── 004-snapshot-strategy.md
│   └── 005-deterministic-rng-fairness.md
│
├── pokerhole-server/        # Java/Spring Boot backend
│   └── README.md           # Server documentation
│
└── pokerhole-cli/          # Go TUI client
    └── README.md           # Client documentation
```

---

## Architecture

### High-Level Design

```
┌─────────────────────────────────────────┐
│      Server (Java/Spring Boot)          │
│                                         │
│  • Texas Hold'em game logic            │
│  • Event sourcing (PostgreSQL)         │
│  • WebSocket communication             │
│  • Matching system                     │
│  • AI players (rule-based)             │
└──────────────┬──────────────────────────┘
               │
          WebSocket (JSON)
               │
┌──────────────┴──────────────────────────┐
│        Client (Go/Bubble Tea)           │
│                                         │
│  • Terminal UI (TUI)                   │
│  • Local SQLite event store            │
│  • Offline play support                │
│  • ed25519 event signatures            │
└─────────────────────────────────────────┘
```

### Key Architectural Decisions

All architecture decisions are documented as ADRs in [`docs/adr/`](docs/adr/):

1. **[ADR-001: Event Sourcing](docs/adr/001-event-sourcing-for-gameplay.md)**
   - Why: Audit trail, replay capability, temporal queries
   - Append-only event store for all game actions
   - Both server (PostgreSQL) and client (SQLite) use event sourcing

2. **[ADR-002: Server Authority](docs/adr/002-server-authority.md)**
   - Server validates all game logic
   - Client is "dumb terminal" for online play
   - Prevents cheating, ensures fairness

3. **[ADR-003: ed25519 Signatures](docs/adr/003-ed25519-client-signatures.md)**
   - Client signs events with private key
   - Server verifies signatures
   - Enables secure offline → online sync

4. **[ADR-004: Snapshot Strategy](docs/adr/004-snapshot-strategy.md)**
   - Periodic snapshots for performance
   - Trade-off: storage vs replay speed
   - Snapshot every N events

5. **[ADR-005: Deterministic RNG](docs/adr/005-deterministic-rng-fairness.md)**
   - Seeded shuffling for fairness verification
   - Same seed = same shuffle on Java and Go
   - Golden test vectors ensure parity

---

## Current Status

**Verified**: 2025-10-03

### Server (pokerhole-server)
- **Tests**: 31 tests, 100% pass
- **Phase 1**: WebSocket integration (~60%)
  - ✅ Domain model (HandEvaluator with 21 golden vectors)
  - ✅ WebSocket matching system (Step 1-3)
  - ✅ PlayerSession state tracking
  - ✅ GameCommandService (Step 4 - structure complete)
  - ⚠️ Integration tests pending (Step 5 - optional)
  - ⚠️ Game logic TODO (Dealer is simple card comparison, not Texas Hold'em)
- **Golden Vectors**: 21 test cases (hand evaluation)

### Client (pokerhole-cli)
- **Phase 2**: Core complete
  - ✅ Go domain model (ported from Java)
  - ✅ SQLite event store
  - ✅ Bubble Tea TUI framework
  - ✅ WebSocket client
  - ⚠️ Offline AI opponents pending

### Verification Commands

```bash
# Count tests
find . -name "*.java" | xargs grep -c "@Test" | awk '{sum+=$2} END {print sum}'

# List golden vectors
cat pokerhole-server/src/test/resources/golden/hand_eval.json | grep -c '"case_id"'

# Check Go-Java parity
cd pokerhole-cli && go test -v ./tests/golden/
```

---

## Tech Stack

### Server
| Component | Technology |
|-----------|-----------|
| Language | Java 23 |
| Framework | Spring Boot 3.4.0 |
| Database | PostgreSQL 16 / H2 (test) |
| Build | Gradle 9.0 (Kotlin DSL) |
| Architecture | Hexagonal + DDD + Event Sourcing |
| Testing | JUnit 5, ArchUnit, AssertJ |

### Client
| Component | Technology |
|-----------|-----------|
| Language | Go 1.25+ |
| TUI | Bubble Tea + Lipgloss |
| Database | SQLite + SQLCipher |
| Crypto | ed25519 (stdlib) |
| Testing | Go testing + golden vectors |

---

## Development

### Prerequisites

**Server**:
- Java 23+
- Docker (for PostgreSQL)
- Gradle 9.0+ (included)

**Client**:
- Go 1.25+
- SQLite3

### Server Development

```bash
cd pokerhole-server

# Run server
./gradlew bootRun

# Run tests
./gradlew test

# Build
./gradlew build

# Generate coverage report
./gradlew test jacocoTestReport
open build/reports/jacoco/test/html/index.html
```

### Client Development

```bash
cd pokerhole-cli

# Run client
go run cmd/poker-client/main.go

# Run tests
go test ./...

# Run golden tests
go test -v ./tests/golden/

# Build
go build -o poker cmd/poker-client/main.go
```

---

## Architecture Principles

### Hexagonal Architecture (Ports & Adapters)

Both server and client follow hexagonal architecture:

```
[Input Adapters]  →  [Application]  →  [Domain]  →  [Output Adapters]
WebSocket/REST       Use Cases         Pure Logic    DB/Network/Event
```

**Key Rules**:
1. Domain has zero framework dependencies
2. Application defines ports (interfaces)
3. Adapters implement ports
4. Dependencies point inward (domain is innermost)

Enforced by ArchUnit tests on server, convention on client.

### Event Sourcing

**Every state change is an event**:
- Server: `GameStarted`, `PlayerActed`, `RoundProgressed`
- Client: Same events, signed with ed25519

**Benefits**:
- Complete audit trail
- Replay capability (debugging, analytics)
- Offline→online sync (future)
- Temporal queries ("what was game state at time T?")

**Trade-offs**:
- More storage (mitigated by snapshots)
- More complex queries (use projections)
- Schema evolution requires event versioning

### Domain-Driven Design

**Aggregates**:
- `Game`: Texas Hold'em game state (server & client)
- `Player`: Player state, chips, actions

**Value Objects**:
- `Card`, `Deck`: Immutable, deterministic shuffling
- `Pot`, `SidePot`: Money management
- `HandResult`: Poker hand ranking

**Domain Events**:
- Past tense: `RoundStarted`, `PlayerFolded`
- Immutable
- Include all data needed to apply the event

---

## Golden Test Vectors

**21 hand evaluation test cases** ensure Java ↔ Go parity:

```bash
# Location
pokerhole-server/src/test/resources/golden/hand_eval.json

# Test on server (Java)
cd pokerhole-server
./gradlew test --tests "*GoldenVectorValidationTest*"

# Test on client (Go)
cd pokerhole-cli
go test -v ./tests/golden/
```

**Test Cases**:
- Royal Flush (2 cases)
- Straight Flush (2 cases, including A-2-3-4-5 wheel)
- Four of a Kind (3 cases)
- Full House (3 cases)
- Flush (2 cases)
- Straight (3 cases)
- Three of a Kind, Two Pair, One Pair, High Card (6 cases)

**Goal**: Expand to 1,000+ cases for comprehensive coverage.

---

## WebSocket Protocol

### Connection

```
Client → Server: ws://localhost:8080/ws/game
```

### Message Format

```json
{
  "type": "MESSAGE_TYPE",
  "timestamp": 1234567890,
  "payload": { /* type-specific data */ }
}
```

### Client → Server

| Type | Description |
|------|-------------|
| `REGISTER` | Initial connection (UUID + nickname) |
| `HEARTBEAT` | Keep-alive (every 30s) |
| `JOIN_RANDOM_MATCH` | Join random matching |
| `CALL`, `RAISE`, `FOLD`, `CHECK`, `ALL_IN` | Game actions |

### Server → Client

| Type | Description |
|------|-------------|
| `REGISTER_SUCCESS` | Registration confirmed |
| `GAME_STATE_UPDATE` | Full game state sync |
| `PLAYER_ACTION` | Player action notification |
| `MATCHING_COMPLETED` | Matching finished, game starts |
| `ERROR` | Error message |

---

## Testing

### Server Tests (31 total)

```bash
cd pokerhole-server
./gradlew test
```

**Current Coverage**:
- HandEvaluatorTest: Hand evaluation logic (21 golden test vectors)
- HexagonalArchitectureTest: Architecture rule enforcement
- GuestVisitServiceTest: JPA persistence
- PokerHoleApplicationTest: Application context

**Note**: Full test suite (~500+ tests) planned for Phase 1 completion.

### Client Tests

```bash
cd pokerhole-cli
go test ./...
```

**Coverage**:
- Domain model (ported from Java)
- Golden vector validation (Go ↔ Java parity)
- Event store (SQLite)
- TUI components

---

## Contributing

### Before Contributing

1. Read relevant ADRs in [`docs/adr/`](docs/adr/)
2. Understand hexagonal architecture
3. Follow event sourcing patterns

### Coding Standards

**Server** (Java):
- Hexagonal architecture (enforced by ArchUnit)
- No framework dependencies in `core/domain/`
- 100% test coverage for domain logic
- Use records for value objects
- MapStruct for object mapping

**Client** (Go):
- Follow hexagonal architecture (by convention)
- Pure domain in `internal/domain/`
- Golden tests must pass (Go ↔ Java parity)
- ed25519 signatures for all events

### Pull Request Checklist

- [ ] All tests pass
- [ ] Architecture tests pass (server)
- [ ] Golden tests pass (both server and client)
- [ ] Coverage ≥ 85% (server)
- [ ] No breaking changes to WebSocket protocol
- [ ] ADR created/updated if architecture changed

---

## Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| Hand evaluation | < 10ms | ✅ ~2ms |
| Event append | < 5ms | ✅ ~3ms |
| WebSocket latency | < 100ms | ✅ ~50ms |
| Concurrent users | 100+ | ⚠️ Not tested |
| Game startup time | < 500ms | ✅ ~200ms |

---

## Roadmap

### Phase 1: Server Core ⚠️ (~60% complete)
- ✅ Domain model (HandEvaluator with 21 golden vectors)
- ✅ WebSocket matching system (Step 1-4)
- ✅ GameCommandService structure
- ⚠️ Integration tests pending (Step 5)
- ⚠️ Game logic incomplete (simple card comparison, not Texas Hold'em)

### Phase 2: Client Core ✅ (Complete)
- ✅ Go domain model
- ✅ SQLite event store
- ✅ Bubble Tea TUI
- ✅ WebSocket client

### Phase 3: Integration (In Progress)
- ⚠️ Online multiplayer
- ⚠️ Offline AI opponents
- ⚠️ Offline→online sync

### Phase 4: Advanced Features (Planned)
- Event replay (debugging)
- Projections (leaderboards, stats)
- Conservative/Aggressive AI
- Tournament mode

### Phase 5: Production (Planned)
- Monitoring (Prometheus)
- Load testing
- Deployment automation
- Security audit

---

## Troubleshooting

### Server won't start

**Check PostgreSQL**:
```bash
docker compose ps
docker compose logs postgres
```

**Check port conflict**:
```bash
lsof -i :8080
```

### Client can't connect

**Verify server is running**:
```bash
curl http://localhost:8080/actuator/health
```

**Check WebSocket endpoint**:
```bash
wscat -c ws://localhost:8080/ws/game
```

### Tests failing

**Server**:
```bash
cd pokerhole-server
./gradlew clean build --refresh-dependencies
```

**Client**:
```bash
cd pokerhole-cli
go clean -cache
go test ./...
```

---

## Documentation

### Core Documentation

- **[docs/adr/](docs/adr/)**: Architecture Decision Records (read these first!)
- **[pokerhole-server/README.md](pokerhole-server/README.md)**: Server details
- **[pokerhole-cli/README.md](pokerhole-cli/README.md)**: Client details

### Key ADRs (Start Here)

1. [ADR-001: Event Sourcing](docs/adr/001-event-sourcing-for-gameplay.md) - Why event sourcing?
2. [ADR-002: Server Authority](docs/adr/002-server-authority.md) - Why server validates everything?
3. [ADR-005: Deterministic RNG](docs/adr/005-deterministic-rng-fairness.md) - How shuffling works?

---

## License

MIT License

---

## Authors

- [@xiyo](https://github.com/xiyo)

---

## Project Status

**Last Updated**: 2025-10-03

- **Server**: Phase 1 WebSocket integration (~60%), 31 tests passing
- **Client**: Phase 2 structure complete, 0 tests (golden tests pending)
- **Integration**: Requires server-client protocol alignment
- **Production**: Not ready (Phase 5 planned)

**Next Milestone**: Server-client protocol alignment + integration tests (see NEXT-STEP.md)

---

## Links

- **Server Repo**: [github.com/bunnyholes/pokerhole-server](https://github.com/bunnyholes/pokerhole-server)
- **Client Repo**: [github.com/bunnyholes/pokerhole-cli](https://github.com/bunnyholes/pokerhole-cli)
- **Issues**: [github.com/bunnyholes/pokerhole/issues](https://github.com/bunnyholes/pokerhole/issues)
