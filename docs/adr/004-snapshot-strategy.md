# ADR-004: Snapshot Strategy at Phase Boundaries and Event Intervals

**Status**: Accepted
**Date**: 2025-10-02
**Authors**: [@xiyo]
**Deciders**: [@xiyo]

---

## Context

Event Sourcing (ADR-001) provides complete audit trail and replay capability, but replaying hundreds of events on every state reconstruction is inefficient:

**Performance Problems**:
- Loading game state requires replaying all events from start
- Long games (100+ events) = slow startup (seconds instead of milliseconds)
- Mobile clients have limited CPU/battery
- Frequent reconnections in unstable networks compound the problem

**Storage Problems**:
- Event logs grow unbounded over time
- 100 games × 200 events/game = 20,000 events to store
- Old games rarely need full event history

**Requirements**:
- Fast state reconstruction (<100ms for typical game)
- Bounded replay cost (constant or logarithmic, not linear)
- Support for archive/compress old games
- Maintain full audit trail (never delete events)

---

## Decision

**We will create snapshots at phase boundaries (after Showdown) and every N=50 events.**

### Snapshot Trigger Rules

1. **Phase Boundary**: After every Showdown (round completion)
   - Captures clean state (pots resolved, cards revealed, winners determined)
   - Natural checkpoint in game flow

2. **Event Count**: Every 50 events since last snapshot
   - Prevents unbounded replay even in long betting rounds
   - Configurable per environment (50 for production, 10 for testing)

3. **Manual**: On-demand via admin API (for debugging, migration)

### Snapshot Structure

```json
{
  "snapshot_id": "uuid-v4",
  "game_id": "uuid-v4",
  "taken_at": "2025-10-02T01:30:00Z",
  "version": 1,
  "last_event_seq": 150,
  "state": {
    "game_phase": "TURN",
    "pot": 2400,
    "side_pots": [],
    "community_cards": ["AS", "KH", "QD", "JC"],
    "players": [
      {
        "player_id": "...",
        "stack": 8500,
        "bet": 200,
        "hole_cards": ["10H", "9H"],
        "status": "ACTIVE",
        "position": 0
      }
    ],
    "dealer_position": 0,
    "to_act": 1,
    "current_bet": 200
  }
}
```

### Replay Algorithm

```python
def load_game_state(game_id):
    # 1. Load latest snapshot
    snapshot = db.get_latest_snapshot(game_id)
    if snapshot is None:
        # No snapshot, replay from beginning
        events = db.get_all_events(game_id)
        state = replay_events(events, from_scratch=True)
    else:
        # Replay events since snapshot
        events = db.get_events_after(game_id, snapshot.last_event_seq)
        state = replay_events(events, from_snapshot=snapshot.state)
    return state
```

### Archive Strategy

After 30 days:
1. **Compress event log**: gzip original events → archive storage
2. **Keep final snapshot**: Last snapshot remains accessible
3. **Create summary**: Single summary event with game outcome
   ```json
   {
     "type": "GameArchived",
     "game_id": "...",
     "completed_at": "...",
     "total_events": 237,
     "winner": "player-123",
     "final_pot": 5000,
     "duration_minutes": 42
   }
   ```

---

## Consequences

### Positive

- **Fast loads**: Replay ≤50 events instead of full game history
- **Predictable performance**: O(1) snapshot + O(k) replay where k≤50
- **Storage efficiency**: Archive old games, keep summary accessible
- **Graceful degradation**: If snapshot corrupted, fall back to full replay
- **Testing friendly**: Snapshots enable "start from middle of game" tests

### Negative

- **Storage overhead**: Each snapshot ~5-10KB (mitigated by compression)
- **Consistency risk**: Snapshot bugs harder to detect than event bugs
- **Complexity**: Must maintain snapshot logic alongside event replay
- **Stale snapshots**: If replay logic changes, old snapshots may be incompatible

### Neutral

- Snapshots are derived state (can always be regenerated from events)
- Trade-off: more frequent snapshots = more storage, less CPU; less frequent = opposite

---

## Alternatives Considered

### Alternative 1: No Snapshots (Pure Event Replay)

**Description**: Always replay full event log from beginning

**Pros**:
- Simplest implementation
- No snapshot consistency issues
- No extra storage

**Cons**:
- Unacceptable performance for long games (>200 events = >1 second load)
- Mobile clients struggle with CPU/battery impact
- Replay time grows linearly with game length

**Why rejected**: Performance unacceptable for production use

### Alternative 2: Only Final Snapshots (One per Game)

**Description**: Create snapshot only when game ends

**Pros**:
- Minimal storage overhead
- Simple logic

**Cons**:
- No benefit for active games (must still replay all events)
- Reconnection still slow
- Doesn't solve in-game performance problem

**Why rejected**: Doesn't address active game performance

### Alternative 3: Per-Event Snapshots (Every Event)

**Description**: Store complete state after every event

**Pros**:
- Instant state reconstruction (no replay)
- Maximum performance

**Cons**:
- Massive storage overhead (10x-50x increase)
- Snapshot consistency becomes critical (can't regenerate easily)
- Defeats event sourcing benefits (becomes CRUD)

**Why rejected**: Storage cost too high, loses event sourcing advantages

### Alternative 4: Time-Based Snapshots (Every 5 Minutes)

**Description**: Create snapshot every N minutes of gameplay

**Pros**:
- Predictable snapshot frequency
- Good for long-running games

**Cons**:
- Poor fit for turn-based game (time between turns varies wildly)
- May snapshot mid-action (unclean state)
- Hard to test (time-dependent)

**Why rejected**: Phase boundaries more natural for poker game flow

---

## Migration Path

### Phase 1: Snapshot Infrastructure (Week 1-2)

**Client (SQLite)**:
```sql
CREATE TABLE snapshots (
  game_id TEXT PRIMARY KEY,
  version INTEGER NOT NULL,
  taken_at INTEGER NOT NULL,
  last_event_seq INTEGER NOT NULL,
  state BLOB NOT NULL, -- gzipped JSON
  state_hash TEXT NOT NULL
);
CREATE INDEX idx_snapshots_taken ON snapshots(taken_at);
```

**Server (PostgreSQL)**:
```sql
CREATE TABLE snapshots (
  game_id UUID PRIMARY KEY,
  version INT NOT NULL,
  taken_at TIMESTAMPTZ NOT NULL,
  last_event_seq BIGINT NOT NULL,
  state BYTEA NOT NULL, -- gzipped JSON
  state_hash TEXT NOT NULL
);
CREATE INDEX idx_snapshots_taken ON snapshots(taken_at);
```

### Phase 2: Snapshot Creation (Week 3)

**Snapshot Triggers**:
```go
// internal/domain/game/snapshot.go
type SnapshotTrigger interface {
    ShouldSnapshot(game *Game, eventCount int) bool
}

type CompositeSnapshotTrigger struct {
    triggers []SnapshotTrigger
}

func (c *CompositeSnapshotTrigger) ShouldSnapshot(game *Game, eventCount int) bool {
    for _, t := range c.triggers {
        if t.ShouldSnapshot(game, eventCount) {
            return true
        }
    }
    return false
}

// Phase boundary trigger
type PhaseBoundaryTrigger struct{}
func (p *PhaseBoundaryTrigger) ShouldSnapshot(game *Game, _ int) bool {
    return game.Phase == SHOWDOWN && game.ShowdownComplete
}

// Event count trigger
type EventCountTrigger struct{ Interval int }
func (e *EventCountTrigger) ShouldSnapshot(_ *Game, count int) bool {
    return count > 0 && count % e.Interval == 0
}
```

### Phase 3: Snapshot Loading & Replay (Week 4)

```go
func (r *GameRepository) LoadGameState(gameID string) (*Game, error) {
    snapshot, err := r.loadLatestSnapshot(gameID)
    if err == ErrSnapshotNotFound {
        // Replay from scratch
        events, _ := r.loadAllEvents(gameID)
        return replayEvents(events, nil)
    }

    // Replay from snapshot
    events, _ := r.loadEventsAfter(gameID, snapshot.LastEventSeq)
    return replayEvents(events, snapshot.State)
}
```

### Phase 4: Archive System (Month 2)

```go
// Background job: archive old games
func ArchiveOldGames(olderThan time.Duration) error {
    cutoff := time.Now().Add(-olderThan) // 30 days ago
    games := db.FindCompletedGamesBefore(cutoff)

    for _, game := range games {
        // 1. Compress event log
        events := db.GetAllEvents(game.ID)
        compressed := gzip.Compress(json.Marshal(events))
        archiveStore.Put(game.ID, compressed)

        // 2. Keep final snapshot
        // (already in snapshots table, no action needed)

        // 3. Create summary event
        summary := createGameSummary(game, events)
        db.InsertEvent(summary)

        // 4. Delete original events (optional, keep if storage allows)
        // db.DeleteEvents(game.ID) // Risky, only if confident
    }
}
```

### Backward Compatibility

- Snapshot version field enables schema evolution
- If snapshot version incompatible: fall back to full event replay
- Snapshot validation: hash check ensures integrity

### Rollback Plan

- If snapshots cause issues:
  - Disable snapshot creation (keep replay-only path)
  - Delete all snapshots, rely on full event replay
  - Exit criteria: Snapshot-related bugs exceed threshold (3+ critical bugs/month)

---

## Related Decisions

- **ADR-001**: Event Sourcing - snapshots optimize event replay
- **ADR-006** (planned): Event Schema Evolution - snapshot versioning strategy

---

## References

- [Snapshots in Event Sourcing](https://www.eventstore.com/blog/snapshots-in-event-sourcing)
- [CQRS Journey: Snapshots](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj591559(v=pandp.10))
- [Event Store Snapshots](https://developers.eventstore.com/server/v21.10/streams/projections/indexing.html)

---

## Notes

### Snapshot Validation

Ensure snapshot integrity:
```go
func validateSnapshot(snap *Snapshot) error {
    // 1. Hash check
    computed := sha256.Sum256(snap.State)
    if hex.EncodeToString(computed[:]) != snap.StateHash {
        return ErrSnapshotCorrupted
    }

    // 2. Deserialization check
    var state GameState
    if err := json.Unmarshal(snap.State, &state); err != nil {
        return ErrSnapshotInvalid
    }

    // 3. Domain invariants check
    if !state.IsValid() {
        return ErrSnapshotInvalidState
    }

    return nil
}
```

### Performance Metrics (Estimates)

**Without snapshots** (200-event game):
- Replay time: ~500ms (2.5ms/event)
- CPU: High (200 event applications)

**With snapshots** (every 50 events):
- Snapshot load: ~10ms
- Replay time: ~125ms (50 events max)
- Total: ~135ms
- **Improvement: 73% faster**

### Storage Impact

Per game (200 events):
- Events: 200 × 0.5KB = 100KB
- Snapshots: 4 × 5KB = 20KB (gzipped)
- **Overhead: 20%**

After archiving (30 days):
- Archived events: 100KB gzipped → 20KB (80% compression)
- Final snapshot: 5KB
- Summary: 0.5KB
- **Total: 25.5KB (74% reduction)**

### Snapshot Frequency Tuning

| Frequency | Replay Max | Snapshot Overhead | Recommended For |
|-----------|-----------|-------------------|-----------------|
| Every 10 events | 10 events | 40KB/game | Development/testing |
| Every 50 events | 50 events | 20KB/game | **Production (default)** |
| Every 100 events | 100 events | 10KB/game | Low-storage environments |
| Phase only | 50-200 events | 5KB/game | Slow betting games |

### Testing Snapshots

```go
func TestSnapshotConsistency(t *testing.T) {
    // 1. Create game, play 100 events
    game := createTestGame()
    events := playRandomEvents(game, 100)

    // 2. Replay from scratch
    state1 := replayAllEvents(events)

    // 3. Create snapshot at event 50, replay from there
    snapshot := createSnapshot(replayEvents(events[:50]))
    state2 := replayEvents(events[50:], snapshot)

    // 4. States must be identical
    assert.Equal(t, state1, state2)
}
```

### Monitoring

Metrics to track:
- Snapshot creation time (p50, p95, p99)
- Snapshot size distribution
- Replay time (with/without snapshot)
- Snapshot corruption rate
- Archive success rate
