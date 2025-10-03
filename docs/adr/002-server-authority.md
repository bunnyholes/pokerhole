# ADR-002: Server as Authoritative Orderer and Validator

**Status**: Accepted
**Date**: 2025-10-02
**Authors**: [@xiyo]
**Deciders**: [@xiyo]

---

## Context

In a hybrid online/offline poker game, we must decide who has final authority over game state:

**Key requirements**:
- **Fairness**: Prevent cheating, ensure rules are enforced consistently
- **Ordering**: Resolve concurrent actions from multiple players
- **Sync**: Merge offline gameplay into authoritative server state
- **Trust**: Players must trust that game outcomes are legitimate

**Challenges**:
- Clients can be compromised (modified binaries, debuggers, network manipulation)
- Network delays create timing ambiguities (who acted first?)
- Offline games have no server to validate
- Players may dispute outcomes

**Trust model**:
- Online games: Server is trusted authority
- Offline games: Client is trusted (but verified on sync)
- Hybrid: Server validates and orders all submitted events

---

## Decision

**The server is the single authoritative orderer and validator for all online gameplay.**

### Core Principles

1. **Server assigns authoritative order**: `server_seq` is the canonical event sequence
2. **Server validates all events**: Domain rules enforced server-side, invalid events rejected
3. **Client events are proposals**: Client sends signed event proposals, server accepts/rejects
4. **Offline events sync as batch**: Server validates entire offline game log on reconnection

### Event Flow

```
┌─────────┐                          ┌─────────┐
│ Client  │                          │ Server  │
└────┬────┘                          └────┬────┘
     │                                    │
     │ Cmd.PlayerAction(sig, client_seq) │
     ├──────────────────────────────────>│
     │                                    │ Validate signature
     │                                    │ Check domain rules
     │                                    │ Assign server_seq
     │                                    │ Append to event log
     │                                    │ Broadcast to all players
     │                                    │
     │<──────────────────────────────────┤ Committed(event_id, server_seq)
     │                                    │
     │<══════════════════════════════════┤ State.Patch(events) [to all]
     │                                    │
```

### Rejected Events

```
┌─────────┐                          ┌─────────┐
│ Client  │                          │ Server  │
└────┬────┘                          └────┬────┘
     │                                    │
     │ Cmd.PlayerAction(RAISE, 9999)     │
     ├──────────────────────────────────>│
     │                                    │ Validate: INSUFFICIENT_STACK
     │                                    │ Reject event
     │                                    │
     │<──────────────────────────────────┤ Rejected(event_id, code, reason)
     │                                    │
     │<──────────────────────────────────┤ State.Replace(snapshot) [to sender]
     │                                    │
```

### Offline Sync Flow

```
┌─────────┐                          ┌─────────┐
│ Client  │                          │ Server  │
└────┬────┘                          └────┬────┘
     │                                    │
     │ SyncRequest(offline_events[0..N]) │
     ├──────────────────────────────────>│
     │                                    │ Verify signatures
     │                                    │ Validate game logic
     │                                    │ Assign server_seq to each
     │                                    │ Append to event log
     │                                    │
     │<──────────────────────────────────┤ SyncResponse(accepted[], rejected[])
     │                                    │
```

---

## Consequences

### Positive

- **Anti-cheat**: Server validates all rules, clients cannot fake actions
- **Deterministic ordering**: `server_seq` eliminates timing ambiguities
- **Fair play**: All players see identical event sequence
- **Dispute resolution**: Server event log is authoritative record
- **Simplicity**: Single source of truth simplifies debugging

### Negative

- **Latency sensitivity**: Network delay affects responsiveness (mitigated by optimistic UI)
- **Server dependency**: Online games require server availability
- **Offline limitations**: Offline games can't be validated until sync
- **Trust concentration**: Players must trust server implementation

### Neutral

- Requires robust server infrastructure (monitoring, HA for v2.0+)
- Client must handle rejected events gracefully (compensation + UI feedback)
- Offline games are "pending validation" until synced

---

## Alternatives Considered

### Alternative 1: Peer-to-Peer Consensus

**Description**: Players reach consensus via distributed algorithm (Raft, Paxos, etc.)

**Pros**:
- No central authority required
- More resilient to single-point failure

**Cons**:
- Extremely complex to implement correctly
- Vulnerable to Sybil attacks (one player runs multiple nodes)
- High latency for consensus rounds
- Mobile/unstable networks make consensus unreliable

**Why rejected**: Complexity outweighs benefits; not suitable for real-time card game

### Alternative 2: Client Authority with Blockchain

**Description**: Use blockchain for tamper-proof event log, clients self-validate

**Pros**:
- Decentralized trust
- Immutable audit trail

**Cons**:
- Massive complexity and cost
- Slow (block confirmation times)
- Energy-intensive
- Doesn't solve cheating (malicious client still cheats locally)

**Why rejected**: Blockchain is overkill and doesn't solve core trust problem

### Alternative 3: Trusted Execution Environment (TEE)

**Description**: Run validation logic in secure enclave (Intel SGX, ARM TrustZone)

**Pros**:
- Hardware-level tamper resistance
- Can validate on client side

**Cons**:
- Platform-specific (not cross-platform)
- Complex attestation flows
- Limited availability on mobile/web
- Requires specialized knowledge

**Why rejected**: Not portable, adds extreme complexity for marginal benefit

---

## Migration Path

### Phase 1: Server-Side Validation Infrastructure (Month 1-2)

1. Implement domain validators for all event types
   - `ValidatePlayerAction`: Check turn order, stack size, legal moves
   - `ValidateDealCards`: Verify deck state, hand counts
   - `ValidateShowdown`: Confirm pot distribution logic

2. Build event ordering system
   - Assign monotonic `server_seq` on commit
   - Detect and reject duplicate `(game_id, client_seq)` pairs

3. Create rejection framework
   - Error codes: `ILLEGAL_TURN`, `INSUFFICIENT_STACK`, `INVALID_SIG`, etc.
   - Rejection response with detailed reason

### Phase 2: Client-Side Optimistic UI (Month 3)

1. Implement local event queue
   - Submit events to server immediately
   - Apply optimistically to local projection
   - Roll back on rejection via compensation event

2. Add rejection handling UI
   - Toast notification: "Action rejected: Not your turn"
   - Auto-refresh game state from server

### Phase 3: Offline Sync Validation (Month 4)

1. Batch validation endpoint
   - Accept array of offline events
   - Validate entire game session atomically
   - Return per-event accept/reject status

2. Signature verification
   - Verify ed25519 signatures on all events
   - Reject events with invalid/missing signatures

### Backward Compatibility

- Initial implementation (v1.0) - no backward compatibility needed
- Future: Server version negotiation via `X-Proto-Version` header
- Graceful degradation: Older clients receive compatibility errors

### Rollback Plan

- If server authority proves problematic:
  - Fallback: Treat server as "advisory validator" only
  - Clients keep local authority, server just logs events
  - Exit criteria: Server performance/reliability issues preventing gameplay

---

## Related Decisions

- **ADR-001**: Event Sourcing - server validates and orders events
- **ADR-003**: ed25519 Signatures - clients sign events, server verifies
- **ADR-005**: RNG Fairness - server controls card shuffling

---

## References

- [Cheating in Online Games](https://www.gamedeveloper.com/programming/building-a-better-anti-cheat-system)
- [Authoritative Server Pattern](https://gabrielgambetta.com/client-server-game-architecture.html)
- [Valve Anti-Cheat (VAC) Architecture](https://partner.steamgames.com/doc/features/anticheat)

---

## Notes

### Future Considerations

- **v1.1**: Add server-side replay verification (detect client-side tampering retroactively)
- **v2.0**: Implement trusted replay logs for tournament verification
- **v3.0**: Explore verifiable shuffle protocols (commit-reveal schemes) for provable fairness

### Edge Cases

- **Network partition during game**: Players receive grace period (90s), AI substitute if disconnected
- **Server crash**: Game state recovered from event log + snapshots (no data loss)
- **Malicious client**: Server rejection prevents invalid state, repeated violations → ban

### Security Guarantees

- Server validates:
  - Signature authenticity (ADR-003)
  - Domain rules (poker logic)
  - Turn order (sequential action enforcement)
  - Stack integrity (chips conservation)

- Server does NOT trust:
  - Client-reported state
  - Client timestamps (server uses `applied_at`)
  - Client-computed outcomes (server recalculates)
