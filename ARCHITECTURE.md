# PokerHole Architecture Storyboard

**목적**: 서버(Java)와 클라이언트(Go)의 헥사고날 아키텍처 "거울" 설계

---

## 1. 헥사고날 아키텍처 전체 구조

```mermaid
graph TB
    subgraph "Server (Java)"
        subgraph "Adapter In (Server)"
            WS_IN[WebSocket Handler]
            REST_IN[REST Controller]
        end

        subgraph "Application (Server)"
            UC_S[Use Cases<br/>GameService<br/>MatchingService]
        end

        subgraph "Domain (Server)"
            DOMAIN_S[Domain Logic<br/>Card, Deck<br/>HandEvaluator<br/>Player, Game]
        end

        subgraph "Adapter Out (Server)"
            DB_OUT[PostgreSQL<br/>Event Store]
            WS_OUT[WebSocket Broadcast]
        end

        WS_IN --> UC_S
        REST_IN --> UC_S
        UC_S --> DOMAIN_S
        UC_S --> DB_OUT
        UC_S --> WS_OUT
    end

    subgraph "Client (Go)"
        subgraph "Adapter In (Client)"
            TUI_IN[Terminal UI<br/>Bubble Tea]
        end

        subgraph "Application (Client)"
            UC_C[Use Cases<br/>GameService<br/>OfflineService]
        end

        subgraph "Domain (Client)"
            DOMAIN_C[Domain Logic<br/>Card, Deck<br/>HandEvaluator<br/>Player, Game]
        end

        subgraph "Adapter Out (Client)"
            WS_C_OUT[WebSocket Client<br/>Online Mode]
            SQLITE_OUT[SQLite Event Store<br/>Offline Mode]
        end

        TUI_IN --> UC_C
        UC_C --> DOMAIN_C
        UC_C --> WS_C_OUT
        UC_C --> SQLITE_OUT
    end

    WS_OUT -.->|Online Mode| WS_C_OUT

    style DOMAIN_S fill:#ff6b6b,color:#fff
    style DOMAIN_C fill:#ff6b6b,color:#fff
    style WS_OUT stroke:#4ecdc4,stroke-width:3px
    style WS_C_OUT stroke:#4ecdc4,stroke-width:3px
```

**핵심 원칙**:
- 🔴 **Domain Layer (빨강)**: 순수 로직, 프레임워크 의존성 ZERO
- 🔵 **Application Layer**: Use Cases, Port 인터페이스
- 🟢 **Adapter Layer**: 외부 세계와의 연결 (UI, DB, Network)

---

## 2. 서버-클라이언트 도메인 매핑 (거울 구조)

```mermaid
graph LR
    subgraph "Java (Server)"
        subgraph "core/domain/card"
            J_Card[Card.java]
            J_Deck[Deck.java]
            J_Suit[Suit.java]
            J_Rank[Rank.java]
        end

        subgraph "core/domain/game"
            J_HE[HandEvaluator.java]
            J_PD[PotDistributor.java]
            J_WR[WinnerResolver.java]
        end

        subgraph "core/domain/game/vo"
            J_HR[HandResult.java]
            J_Pot[Pot.java]
            J_BR[BettingRound.java]
        end

        subgraph "core/domain/player"
            J_Player[Player.java]
        end
    end

    subgraph "Go (Client)"
        subgraph "core/domain/card"
            G_Card[card.go]
            G_Deck[deck.go]
            G_Suit[suit.go]
            G_Rank[rank.go]
        end

        subgraph "core/domain/game"
            G_HE[hand_evaluator.go]
            G_PD[pot_distributor.go]
            G_WR[winner_resolver.go]
        end

        subgraph "core/domain/game/vo"
            G_HR[hand_result.go]
            G_Pot[pot.go]
            G_BR[betting_round.go]
        end

        subgraph "core/domain/player"
            G_Player[player.go]
        end
    end

    J_Card <-.->|거울| G_Card
    J_Deck <-.->|거울| G_Deck
    J_Suit <-.->|거울| G_Suit
    J_Rank <-.->|거울| G_Rank
    J_HE <-.->|거울| G_HE
    J_PD <-.->|거울| G_PD
    J_WR <-.->|거울| G_WR
    J_HR <-.->|거울| G_HR
    J_Pot <-.->|거울| G_Pot
    J_BR <-.->|거울| G_BR
    J_Player <-.->|거울| G_Player

    style J_Card fill:#ffe66d
    style G_Card fill:#ffe66d
    style J_HE fill:#a8e6cf
    style G_HE fill:#a8e6cf
```

---

## 3. 온라인 vs 오프라인 모드 플로우

### 3.1 온라인 모드 (Server Authority)

```mermaid
sequenceDiagram
    participant User
    participant TUI as TUI (Adapter In)
    participant UseCase as GameService (Application)
    participant Domain as HandEvaluator (Domain)
    participant WS as WebSocket (Adapter Out)
    participant Server as Java Server

    User->>TUI: RAISE 100
    TUI->>UseCase: ExecuteAction(RAISE, 100)

    Note over UseCase: 클라이언트 즉시 검증<br/>(UX 개선)
    UseCase->>Domain: ValidateAction(RAISE, 100)
    Domain-->>UseCase: OK (or Error)

    alt 클라 검증 성공
        UseCase->>WS: Send(PlayerActed event)
        WS->>Server: WebSocket Message

        Note over Server: 서버 최종 검증<br/>(Server Authority)
        Server-->>WS: GameStateUpdate
        WS-->>UseCase: Event Received
        UseCase->>Domain: ApplyEvent(PlayerActed)
        Domain-->>UseCase: New State
        UseCase-->>TUI: Update UI
        TUI-->>User: Show Result
    else 클라 검증 실패
        UseCase-->>TUI: Show Error (즉시)
        TUI-->>User: Invalid Action
    end
```

### 3.2 오프라인 모드 (Client Authority)

```mermaid
sequenceDiagram
    participant User
    participant TUI as TUI (Adapter In)
    participant UseCase as OfflineGameService
    participant Domain as HandEvaluator (Domain)
    participant Store as SQLite Event Store

    User->>TUI: RAISE 100
    TUI->>UseCase: ExecuteAction(RAISE, 100)

    UseCase->>Domain: ValidateAction(RAISE, 100)
    Domain-->>UseCase: OK

    UseCase->>Domain: ApplyAction(RAISE, 100)
    Domain-->>UseCase: PlayerActed Event

    UseCase->>Store: Append(PlayerActed)
    Store-->>UseCase: Saved

    UseCase-->>TUI: Update UI
    TUI-->>User: Show Result

    Note over Domain,Store: 클라이언트가 서버 역할도 수행<br/>같은 도메인 로직 사용
```

---

## 4. 패키지 구조 상세 (거울 매핑)

### 4.1 서버 (Java)

```
pokerhole-server/src/main/java/dev/xiyo/pokerhole/
│
├── core/
│   ├── domain/                          # 순수 도메인 (NO Spring)
│   │   ├── card/
│   │   │   ├── Card.java               # Value Object
│   │   │   ├── Deck.java               # Aggregate
│   │   │   ├── Suit.java               # Enum
│   │   │   ├── Rank.java               # Enum
│   │   │   └── Hand.java               # Value Object
│   │   │
│   │   ├── game/
│   │   │   ├── HandEvaluator.java      # Interface
│   │   │   ├── HandEvaluatorImpl.java  # Implementation
│   │   │   ├── PotDistributor.java     # Domain Service
│   │   │   ├── WinnerResolver.java     # Domain Service
│   │   │   ├── GameId.java             # Value Object
│   │   │   ├── GameState.java          # Enum
│   │   │   │
│   │   │   ├── vo/
│   │   │   │   ├── HandResult.java     # Value Object
│   │   │   │   ├── Pot.java            # Value Object
│   │   │   │   ├── SidePot.java        # Value Object
│   │   │   │   ├── BettingRound.java   # Enum
│   │   │   │   └── PlayerAction.java   # Enum
│   │   │   │
│   │   │   └── event/
│   │   │       ├── GameEvent.java      # Interface
│   │   │       ├── RoundStarted.java   # Event
│   │   │       ├── PlayerActed.java    # Event
│   │   │       ├── RoundProgressed.java# Event
│   │   │       └── RoundEnded.java     # Event
│   │   │
│   │   ├── player/
│   │   │   ├── Player.java             # Aggregate
│   │   │   ├── PlayerRecord.java       # Entity
│   │   │   └── vo/
│   │   │       ├── PlayerId.java       # Value Object
│   │   │       ├── Nickname.java       # Value Object
│   │   │       └── PlayerStatus.java   # Enum
│   │   │
│   │   └── shared/
│   │       ├── DomainEvent.java        # Interface
│   │       └── AggregateRoot.java      # Base Class
│   │
│   └── application/                     # Use Cases
│       ├── port/
│       │   ├── in/                     # Driving Ports
│       │   │   └── game/
│       │   │       ├── StartGameUseCase.java
│       │   │       └── ExecuteActionUseCase.java
│       │   │
│       │   └── out/                    # Driven Ports
│       │       ├── EventPublisher.java
│       │       └── GameRepositoryPort.java
│       │
│       └── service/
│           ├── GameCommandService.java
│           └── MatchingService.java
│
└── adapter/                             # Adapters
    ├── in/                             # Driving Adapters
    │   ├── websocket/
    │   │   └── GameWebSocketHandler.java
    │   └── rest/
    │       └── GameController.java
    │
    └── out/                            # Driven Adapters
        ├── persistence/
        │   └── JpaGameRepository.java
        └── event/
            └── SpringEventPublisher.java
```

### 4.2 클라이언트 (Go) - 목표 구조

```
pokerhole-cli/
│
├── cmd/
│   └── poker-client/
│       └── main.go                     # Entry Point
│
└── internal/
    ├── core/
    │   ├── domain/                     # 순수 도메인 (NO 외부 의존성)
    │   │   ├── card/
    │   │   │   ├── card.go            # ← Card.java 거울
    │   │   │   ├── deck.go            # ← Deck.java 거울
    │   │   │   ├── suit.go            # ← Suit.java 거울
    │   │   │   ├── rank.go            # ← Rank.java 거울
    │   │   │   └── hand.go            # ← Hand.java 거울
    │   │   │
    │   │   ├── game/
    │   │   │   ├── hand_evaluator.go       # ← HandEvaluator.java 거울
    │   │   │   ├── hand_evaluator_impl.go  # ← HandEvaluatorImpl.java 거울
    │   │   │   ├── pot_distributor.go      # ← PotDistributor.java 거울
    │   │   │   ├── winner_resolver.go      # ← WinnerResolver.java 거울
    │   │   │   ├── game_id.go              # ← GameId.java 거울
    │   │   │   ├── game_state.go           # ← GameState.java 거울
    │   │   │   │
    │   │   │   ├── vo/
    │   │   │   │   ├── hand_result.go      # ← HandResult.java 거울
    │   │   │   │   ├── pot.go              # ← Pot.java 거울
    │   │   │   │   ├── side_pot.go         # ← SidePot.java 거울
    │   │   │   │   ├── betting_round.go    # ← BettingRound.java 거울
    │   │   │   │   └── player_action.go    # ← PlayerAction.java 거울
    │   │   │   │
    │   │   │   └── event/
    │   │   │       ├── game_event.go       # ← GameEvent.java 거울
    │   │   │       ├── round_started.go    # ← RoundStarted.java 거울
    │   │   │       ├── player_acted.go     # ← PlayerActed.java 거울
    │   │   │       ├── round_progressed.go # ← RoundProgressed.java 거울
    │   │   │       └── round_ended.go      # ← RoundEnded.java 거울
    │   │   │
    │   │   ├── player/
    │   │   │   ├── player.go               # ← Player.java 거울
    │   │   │   ├── player_record.go        # ← PlayerRecord.java 거울
    │   │   │   └── vo/
    │   │   │       ├── player_id.go        # ← PlayerId.java 거울
    │   │   │       ├── nickname.go         # ← Nickname.java 거울
    │   │   │       └── player_status.go    # ← PlayerStatus.java 거울
    │   │   │
    │   │   └── shared/
    │   │       ├── domain_event.go         # ← DomainEvent.java 거울
    │   │       └── aggregate_root.go       # ← AggregateRoot.java 거울
    │   │
    │   └── application/                    # Use Cases
    │       ├── port/
    │       │   ├── in/                     # Driving Ports
    │       │   │   └── game/
    │       │   │       ├── start_game.go
    │       │   │       └── execute_action.go
    │       │   │
    │       │   └── out/                    # Driven Ports
    │       │       ├── event_publisher.go
    │       │       └── game_repository.go
    │       │
    │       └── service/
    │           ├── online_game_service.go   # 온라인 모드
    │           └── offline_game_service.go  # 오프라인 모드
    │
    └── adapter/                            # Adapters
        ├── in/                             # Driving Adapters
        │   └── tui/
        │       ├── game_view.go            # Bubble Tea UI
        │       ├── lobby_view.go
        │       └── model.go
        │
        └── out/                            # Driven Adapters
            ├── websocket/
            │   └── client.go               # 온라인 모드
            └── storage/
                ├── sqlite_event_store.go   # 오프라인 모드
                └── migrations/
```

---

## 5. Golden Test 매핑

```mermaid
graph TB
    subgraph "Test Vectors (공유)"
        JSON[hand_eval.json<br/>21 test cases]
    end

    subgraph "Java Tests"
        J_TEST[GoldenVectorValidationTest.java]
        J_TEST -->|reads| JSON
        J_TEST -->|calls| J_HE[HandEvaluatorImpl.java]
    end

    subgraph "Go Tests"
        G_TEST[hand_eval_test.go]
        G_TEST -->|reads| JSON
        G_TEST -->|calls| G_HE[hand_evaluator_impl.go]
    end

    JSON -.->|Royal Flush| CASE1[AS KS QS JS TS<br/>Expected: ROYAL_FLUSH]
    JSON -.->|Four of a Kind| CASE2[AD AC AH AS 2D<br/>Expected: FOUR_OF_A_KIND]

    style JSON fill:#ffd93d,stroke:#333,stroke-width:3px
    style J_TEST fill:#6bcf7f
    style G_TEST fill:#6bcf7f
```

**검증 원칙**:
- 같은 JSON 파일을 양쪽에서 읽음
- 같은 입력 → 같은 출력 (언어 무관)
- 하나라도 다르면 테스트 실패

---

## 6. 이벤트 소싱 플로우

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant EventStore as PostgreSQL<br/>Event Store
    participant Projection as Game State<br/>Projection

    Note over Client,Server: Game Start
    Client->>Server: START_TEXAS_HOLDEM
    Server->>EventStore: Append(GameStarted)
    Server->>EventStore: Append(RoundStarted)
    Server->>Projection: Replay Events
    Projection-->>Server: Current State
    Server-->>Client: GAME_STATE_UPDATE

    Note over Client,Server: Player Action
    Client->>Server: RAISE 100
    Server->>EventStore: Append(PlayerActed)
    Server->>Projection: Apply Event
    Projection-->>Server: New State
    Server-->>Client: PLAYER_ACTION

    Note over Client,Server: Round Progression
    Server->>EventStore: Append(RoundProgressed)
    Server->>Projection: Apply Event
    Projection-->>Server: New State
    Server-->>Client: ROUND_PROGRESSED

    Note over EventStore: 모든 이벤트 불변 저장
    Note over Projection: 현재 상태 = 이벤트 재생 결과
```

---

## 7. 타입 매핑 가이드

| 개념 | Java | Go | 비고 |
|------|------|----|------|
| **불변 값 객체** | `record` 또는 `final class` | `struct` (unexported fields) | |
| **열거형** | `enum` | `type + const + iota` | |
| **인터페이스** | `interface` | `interface` | Go는 암시적 구현 |
| **에러 처리** | `throw Exception` | `return error` | |
| **Optional** | `Optional<T>` | `*T` (nullable) 또는 `(T, error)` | |
| **리스트** | `List<T>` | `[]T` (slice) | |
| **맵** | `Map<K, V>` | `map[K]V` | |
| **Getter** | `getSuit()` | `Suit()` | Go는 Get 접두사 생략 |
| **생성자** | `new Card(suit, rank)` | `NewCard(suit, rank)` | |

---

## 8. 구현 우선순위

```mermaid
graph TD
    START[시작] --> P1

    subgraph "Phase 1: 기본 도메인"
        P1[card 패키지<br/>Card, Suit, Rank, Deck]
        P1 --> P1_TEST[Golden Test: Deck Shuffle]
    end

    P1_TEST --> P2

    subgraph "Phase 2: 핵심 로직"
        P2[game 패키지<br/>HandEvaluator, HandResult]
        P2 --> P2_TEST[Golden Test: 21 Cases]
    end

    P2_TEST --> P3

    subgraph "Phase 3: 게임 규칙"
        P3[game 패키지<br/>PotDistributor, WinnerResolver]
        P3 --> P3_TEST[Integration Test]
    end

    P3_TEST --> P4

    subgraph "Phase 4: 이벤트"
        P4[event 패키지<br/>PlayerActed, RoundProgressed]
        P4 --> P4_TEST[Event Replay Test]
    end

    P4_TEST --> P5

    subgraph "Phase 5: 어댑터"
        P5[adapter/out<br/>WebSocket + SQLite]
        P5 --> P5_TEST[E2E Test]
    end

    P5_TEST --> END[완료]

    style P1 fill:#ffe66d
    style P2 fill:#ff6b6b,color:#fff
    style P3 fill:#4ecdc4
    style P4 fill:#95e1d3
    style P5 fill:#a8e6cf
```

---

## 9. 핵심 원칙 요약

### 9.1 의존성 규칙

```mermaid
graph TD
    ADAPTER_IN[Adapter In<br/>WebSocket, TUI] -->|의존| APP
    APP[Application<br/>Use Cases] -->|의존| DOMAIN
    DOMAIN[Domain<br/>순수 로직]
    APP -->|의존| ADAPTER_OUT[Adapter Out<br/>DB, WebSocket]

    DOMAIN -.->|절대 의존 금지| APP
    DOMAIN -.->|절대 의존 금지| ADAPTER_IN
    DOMAIN -.->|절대 의존 금지| ADAPTER_OUT

    style DOMAIN fill:#ff6b6b,color:#fff,stroke:#333,stroke-width:4px
    style APP fill:#4ecdc4
    style ADAPTER_IN fill:#a8e6cf
    style ADAPTER_OUT fill:#a8e6cf
```

**규칙**:
- ✅ Domain은 **아무것도 의존하지 않음**
- ✅ Application은 Domain만 의존
- ✅ Adapter는 Application과 Domain 의존 가능
- ❌ Domain이 Framework/Library 의존 = 아키텍처 위반

### 9.2 거울 원칙

| 항목 | 서버 (Java) | 클라 (Go) | 동일해야 하는 것 |
|------|------------|----------|-----------------|
| 개념 | ✅ | ✅ | 타입명, 역할, 책임 |
| 구조 | ✅ | ✅ | 패키지 계층, 파일 위치 |
| 인터페이스 | ✅ | ✅ | 공개 메서드 시그니처 |
| 결과 | ✅ | ✅ | Golden Test 출력값 |
| 구현 | ❌ | ❌ | 언어별 이디엄 다름 |

---

## 10. 다음 단계

### 10.1 즉시 시작 가능

1. **card 패키지 구현** (Go)
   - `suit.go`, `rank.go`, `card.go`, `deck.go`
   - Java 코드 보면서 번역
   - Golden Test 작성 (Deck Shuffle)

2. **HandEvaluator 구현** (Go)
   - `hand_evaluator.go`, `hand_result.go`
   - 21개 Golden Test 벡터 검증

3. **오프라인 모드 구현**
   - SQLite Event Store
   - 이벤트 재생 로직
   - TUI 연결

### 10.2 검증 기준

| Phase | 검증 방법 | 합격 기준 |
|-------|----------|----------|
| Phase 1 | Deck Shuffle Golden Test | Java와 같은 시드 → 같은 카드 순서 |
| Phase 2 | HandEvaluator 21 Cases | 모든 테스트 통과 (100%) |
| Phase 3 | Pot Distribution Test | Side Pot 계산 일치 |
| Phase 4 | Event Replay Test | 같은 이벤트 스트림 → 같은 최종 상태 |
| Phase 5 | E2E Test | 온라인/오프라인 모두 게임 완료 |

---

## 참고 자료

- **CLAUDE.md**: 작업 규칙, 컨벤션
- **ROADMAP.md**: 현재 상태, 다음 단계
- **ADR-001**: Event Sourcing 설계 결정
- **ADR-002**: Server Authority 설계 결정
- **ADR-005**: Deterministic RNG 설계 결정

---

**Last Updated**: 2025-10-04
**Status**: Architecture Design Complete, Ready for Implementation
