# ADR-003: ed25519 Signatures for Client Events

**Status**: Accepted
**Date**: 2025-10-02
**Authors**: [@xiyo]
**Deciders**: [@xiyo]

---

## Context

PokerHole requires cryptographic signatures on client events to ensure:

**Integrity**: Events haven't been tampered with during transit or storage
**Authentication**: Events genuinely originate from claimed player
**Non-repudiation**: Players cannot deny actions they performed
**Offline support**: Events signed offline must be verifiable later

**Requirements**:
- Cross-platform (Go client, Java server)
- Fast signing/verification (low latency)
- Small signature size (network efficiency)
- Secure against modern attacks
- Easy key management

**Constraints**:
- Mobile/desktop clients (varying compute power)
- Potentially thousands of events per game session
- Must work offline (no online key server)
- Keys stored on device (must protect at-rest)

---

## Decision

**We will use ed25519 (EdDSA on Curve25519) for all client event signatures.**

### Key Management

1. **Per-device keypair**: Each client device generates unique ed25519 keypair on first run
2. **Private key storage**: OS-specific secure keychain/keystore
   - macOS: Keychain
   - Linux: Secret Service API (libsecret)
   - Windows: DPAPI (Data Protection API)
3. **Public key registration**: On first connection, client registers public key with server
4. **Multi-device support**: Server stores multiple public keys per player_id

### Signature Format

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "schema_version": "1.0.0",
  "game_id": "...",
  "player_id": "...",
  "type": "PlayerAction",
  "payload": {"action": "RAISE", "amount": 200},
  "client_seq": 42,
  "ts": "2025-10-02T01:23:45Z",
  "sig": "base64(ed25519_signature)"
}
```

**Signed Data**: Canonical JSON of all fields except `sig` itself
```
{event_id}{schema_version}{game_id}{player_id}{type}{payload}{client_seq}{ts}
```

### Verification Flow

```
Server receives event:
1. Extract public_key from player_keys table (player_id, device_id)
2. Reconstruct canonical signed data (all fields except sig)
3. Verify signature using ed25519.Verify(public_key, data, signature)
4. Reject if verification fails (INVALID_SIG error code)
```

---

## Consequences

### Positive

- **Performance**: ed25519 is extremely fast (~70K signatures/sec, ~20K verifications/sec)
- **Small signatures**: 64 bytes (vs 512+ for RSA-2048)
- **Modern crypto**: Resistant to timing attacks, safe curve choice
- **Broad support**: Well-supported libraries (Go: crypto/ed25519, Java: Tink, BouncyCastle)
- **Deterministic**: Same input always produces same signature (simplifies testing)
- **Offline-friendly**: No need for online CA or key server

### Negative

- **Key loss**: Lost device = lost keypair (mitigated by recovery mechanism in ADR-007)
- **OS keychain dependency**: Platform-specific code required
- **Revocation complexity**: No built-in revocation (manual server-side blocklist)
- **Quantum vulnerability**: EdDSA broken by quantum computers (post-quantum migration in v3.0+)

### Neutral

- Requires careful canonical JSON serialization (field ordering, whitespace)
- Server must track device_id → public_key mapping
- Signature verification adds latency (~50μs per event)

---

## Alternatives Considered

### Alternative 1: HMAC with Shared Secret

**Description**: Symmetric HMAC-SHA256 with pre-shared secret per player

**Pros**:
- Simpler implementation (no public/private key pairs)
- Very fast (~100K operations/sec)
- Smaller overhead

**Cons**:
- Shared secret on client (vulnerable if device compromised)
- No non-repudiation (server could forge client events)
- Key distribution problem (how to securely deliver secret?)
- Multi-device requires multiple secrets

**Why rejected**: Insufficient security (symmetric keys on client are risky), no non-repudiation

### Alternative 2: RSA-2048 Signatures

**Description**: Traditional RSA signatures with 2048-bit keys

**Pros**:
- Well-established, widely trusted
- Strong security guarantees
- Hardware acceleration on some platforms

**Cons**:
- Slow (10x slower than ed25519)
- Large signatures (256 bytes vs 64 for ed25519)
- Large keys (slower transmission, storage)
- Timing attack vulnerabilities if not implemented carefully

**Why rejected**: Performance and size overhead unacceptable for high-frequency events

### Alternative 3: ECDSA (secp256k1 or P-256)

**Description**: Elliptic Curve Digital Signature Algorithm

**Pros**:
- Similar performance to ed25519
- Widely used (Bitcoin uses secp256k1)

**Cons**:
- Non-deterministic (requires good RNG, risky on some platforms)
- More complex implementation (k-value nonce generation is error-prone)
- secp256k1 less standard than P-256, P-256 less safe than Curve25519

**Why rejected**: ed25519 is safer (deterministic, safer curve) and simpler

---

## Migration Path

### Phase 1: Key Generation and Storage (Week 1-2)

**Client (Go)**:
```go
// internal/crypto/keypair.go
func GenerateOrLoadKeypair(deviceID string) (ed25519.PrivateKey, ed25519.PublicKey, error) {
    // Try to load from OS keychain
    privKey, err := keychain.Load(deviceID)
    if err == keychain.ErrNotFound {
        // Generate new keypair
        pubKey, privKey, _ := ed25519.GenerateKey(rand.Reader)
        // Store in OS keychain
        keychain.Store(deviceID, privKey)
        return privKey, pubKey, nil
    }
    return privKey, privKey.Public().(ed25519.PublicKey), nil
}
```

**Server (Java)**:
```java
// adapter/out/persistence/jpa/entity/PlayerKeyEntity.java
@Entity
@Table(name = "player_keys")
public class PlayerKeyEntity {
    @EmbeddedId
    private PlayerKeyId id; // (player_id, device_id)

    @Column(nullable = false)
    private byte[] publicKey;

    @Column(nullable = false)
    private Instant createdAt;

    private Instant lastUsedAt;
}
```

### Phase 2: Signature Generation (Week 3)

**Client**:
```go
func SignEvent(event *GameEvent, privKey ed25519.PrivateKey) error {
    canonical := canonicalJSON(event) // All fields except sig
    signature := ed25519.Sign(privKey, []byte(canonical))
    event.Sig = base64.StdEncoding.EncodeToString(signature)
    return nil
}
```

### Phase 3: Verification (Week 4)

**Server**:
```java
public boolean verifyEventSignature(GameEvent event, PublicKey publicKey) {
    String canonical = canonicalJSON(event);
    byte[] signature = Base64.getDecoder().decode(event.getSig());
    return Ed25519.verify(publicKey, canonical.getBytes(UTF_8), signature);
}
```

### Phase 4: Integration (Week 5-6)

1. WebSocket handshake: client sends public key, server stores
2. Every client event: sign before sending
3. Server validates signature before processing
4. Reject with `INVALID_SIG` error code on failure

### Backward Compatibility

- Initial v1.0: All events must be signed (no backward compatibility)
- Future: `schema_version` field allows signature format changes
- Migration: Server can support multiple signature schemes simultaneously

### Rollback Plan

- If ed25519 proves problematic (e.g., platform support issues):
  - Fallback to HMAC-SHA256 with device-specific shared secret
  - Server generates and securely delivers HMAC key on first connection
  - Exit criteria: <95% of platforms support ed25519 keychain storage

---

## Related Decisions

- **ADR-001**: Event Sourcing - events must be tamper-proof
- **ADR-002**: Server Authority - server verifies signatures before accepting events
- **ADR-007** (planned): Account Recovery - handling lost keypairs

---

## References

- [ed25519 Specification (RFC 8032)](https://datatracker.ietf.org/doc/html/rfc8032)
- [Things that use ed25519](https://ianix.com/pub/ed25519-deployment.html)
- [Go crypto/ed25519](https://pkg.go.dev/crypto/ed25519)
- [Tink Crypto Library (Google)](https://github.com/google/tink)
- [Why EdDSA over ECDSA](https://blog.mozilla.org/warner/2011/11/29/ed25519-keys/)

---

## Notes

### Security Considerations

1. **Timing attacks**: ed25519 is designed to be constant-time (resistant)
2. **RNG quality**: Go's crypto/rand and Java's SecureRandom are both suitable
3. **Key derivation**: Do NOT derive keypair from password (use dedicated key gen)
4. **Canonical JSON**: Must use strict field ordering to prevent signature bypass

### Canonical JSON Rules

```
- Fields sorted alphabetically by key
- No whitespace (compact encoding)
- UTF-8 encoding
- Numbers without leading zeros
- Strings with escaped control characters
```

Example:
```json
{"client_seq":42,"event_id":"550e8400...","game_id":"...","payload":{"action":"RAISE","amount":200},"player_id":"...","schema_version":"1.0.0","ts":"2025-10-02T01:23:45Z","type":"PlayerAction"}
```

### Implementation Libraries

**Go (Client)**:
- `crypto/ed25519` (standard library)
- `github.com/99designs/keyring` (OS keychain abstraction)

**Java (Server)**:
- `com.google.crypto.tink:tink` (Google's crypto library)
- `org.bouncycastle:bcprov-jdk18on` (alternative, more features)

### Performance Benchmarks (estimates)

```
Signature generation: ~15μs per event
Verification: ~50μs per event
Network overhead: 64 bytes per event
Key generation: ~1ms (one-time)
```

For 100 events/game:
- Client signing time: 1.5ms total
- Server verification: 5ms total
- Network: 6.4KB signature data

**Acceptable overhead for security guarantees.**

### Multi-Device Scenario

Player with 3 devices (phone, laptop, desktop):
1. Each device generates unique keypair
2. Server stores 3 public keys for player_id
3. On event verification: server tries all registered keys until one succeeds
4. `last_used_at` updated on successful verification (helps identify stale keys)

### Key Rotation (Future)

- v1.0: No rotation (keys are permanent)
- v1.1: Add key rotation API (generate new keypair, register, deprecate old)
- v2.0: Automatic rotation every 180 days with grace period
