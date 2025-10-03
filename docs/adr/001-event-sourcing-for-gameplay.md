# ADR-001: Adopt Event Sourcing for Gameplay State

**Status**: Accepted
**Date**: 2025-10-02
**Authors**: [@xiyo]
**Deciders**: [@xiyo]

---

## Context

PokerHole requires a robust state management strategy that supports:

- **Offline-first architecture**: Players must be able to play standalone games without network connectivity
- **Auditability**: All game actions must be traceable for fairness verification and dispute resolution
- **Replay capability**: Ability to reconstruct any game state at any point in time
- **Sync-friendliness**: Offline games must merge seamlessly with online server state
- **Multi-platform consistency**: Same game logic across Go client and Java server

Traditional CRUD-based state management creates challenges:
- Difficult to merge conflicting offline/online states
- Limited audit trail (only current state, not history)
- Hard to implement undo/replay features
- Synchronization requires complex conflict resolution logic

---

## Decision

**We will use Event Sourcing as the primary state management pattern for all gameplay.**

All game state changes will be represented as immutable events in an append-only log. Current state is derived by replaying events from the beginning or from the latest snapshot.

### Event Structure

```json
{
  "event_id": "uuid-v4",
  "schema_version": "1.0.0",
  "game_id": "uuid-v4",
  "player_id": "client-uuid",
  "type": "PlayerAction",
  "payload": {"action": "RAISE", "amount": 200},
  "client_seq": 42,
  "ts": "2025-10-02T01:23:45Z",
  "sig": "base64(ed25519_signature)"
}
```

### Server-Added Metadata

```json
{
  "server_seq": 1337,
  "applied_at": "2025-10-02T01:23:46Z",
  "validator_version": "1.0.0"
}
```

### Core Principles

1. **Events are immutable**: Once written, never modified
2. **Append-only**: Events are only added to the log
3. **Ordered**: Both client_seq (client-side) and server_seq (server-side) provide ordering
4. **Signed**: All events carry ed25519 signatures for integrity
5. **Versioned**: Schema versioning enables evolution without breaking old events

---

## Consequences

### Positive

- **Complete audit trail**: Every game action is permanently recorded
- **Time travel debugging**: Replay events to any point for debugging
- **Sync-friendly**: Events naturally merge (server assigns authoritative order)
- **Testability**: Deterministic replay enables thorough testing
- **Analytics-ready**: Event stream is perfect for analytics and ML training data
- **Undo/replay**: Easy to implement game replay and analysis features

### Negative

- **Storage overhead**: Events accumulate over time (mitigated by snapshots and archiving)
- **Query complexity**: Deriving current state requires event replay (mitigated by projections)
- **Learning curve**: Team must understand event sourcing patterns
- **Schema evolution complexity**: Must handle old event formats (solved by upcasters)

### Neutral

- Requires snapshot strategy for performance (see ADR-004)
- Necessitates projection layer for read-optimized views
- Event store becomes single source of truth (CQRS pattern)

---

## Alternatives Considered

### Alternative 1: CRUD with Audit Log

**Description**: Traditional relational tables with separate audit log table

**Pros**:
- Familiar pattern to most developers
- Simpler queries for current state
- Well-established ORM support

**Cons**:
- Audit log often incomplete or becomes inconsistent
- Offline-online sync requires complex merge logic
- Hard to guarantee referential integrity between state and audit
- Limited replay capability

**Why rejected**: Insufficient for offline-first requirements and audit needs

### Alternative 2: CQRS without Event Sourcing

**Description**: Command-Query Responsibility Segregation with state-based sync

**Pros**:
- Separates read/write concerns
- Easier to query current state

**Cons**:
- Still lacks complete audit trail
- Sync conflicts harder to resolve without event history
- No time-travel capability

**Why rejected**: Missing event sourcing benefits while adding CQRS complexity

### Alternative 3: Operational Transform (OT) / CRDT

**Description**: Conflict-free replicated data types for distributed state

**Pros**:
- Strong eventual consistency guarantees
- Automatic conflict resolution

**Cons**:
- Extremely complex for poker game logic
- Hard to reason about with business rules
- Limited tooling support
- Server authority model doesn't fit well

**Why rejected**: Overkill complexity, server-authoritative model is simpler

---

## Migration Path

### Phase 1: Core Infrastructure (Month 1-2)

1. Define GameEvent sealed interface (Go) and sealed class (Java)
2. Implement event store adapters (SQLite for client, PostgreSQL for server)
3. Create serialization/deserialization codecs
4. Build basic event replay engine

### Phase 2: Domain Events (Month 3)

1. Define all event types: PlayerJoined, DealtCard, PlayerAction, RoundAdvanced, etc.
2. Implement event validators
3. Create projection builders (GameState, PlayerStats, etc.)

### Phase 3: Integration (Month 4)

1. Wire event store to command handlers
2. Implement sync protocol (client → server event submission)
3. Add signature verification
4. Build snapshot system (ADR-004)

### Backward Compatibility

- Initial version (1.0.0) starts fresh - no backward compatibility needed
- Future schema changes will use upcaster pattern (ADR-006, planned)
- `schema_version` field enables gradual migration

### Rollback Plan

- If event sourcing proves problematic before v1.0 launch:
  - Pivot to state snapshots as primary storage
  - Keep events as audit-only log
  - Use snapshots for sync instead of events
- Exit criteria: Unable to achieve acceptable replay performance after optimization

---

## Related Decisions

- **ADR-002**: Server Authority - events validated and ordered by server
- **ADR-004**: Snapshot Strategy - performance optimization via snapshots
- **ADR-006** (planned): Event Schema Evolution - handling version changes

---

## References

- [Event Sourcing - Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Event Sourcing pattern - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [CQRS Journey - Microsoft](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10))
- [Effective Event Sourcing - Greg Young](https://www.youtube.com/watch?v=JHGkaShoyNs)

---

## Notes

- Event store will be the foundation for future features:
  - Tournament replay and analysis
  - AI training data collection
  - Fraud detection via pattern analysis
  - Leaderboard and statistics
- Consider using [EventStore](https://www.eventstore.com/) as managed solution for v2.0+ if needed
- Projections will be rebuilt from events during schema migrations
