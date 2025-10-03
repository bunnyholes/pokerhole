# ADR-005: Deterministic RNG and Shuffle Fairness Policy

**Status**: Accepted
**Date**: 2025-10-02
**Authors**: [@xiyo]
**Deciders**: [@xiyo]

---

## Context

Card shuffling is the foundation of poker fairness. Players must trust that:

**Randomness**: Cards are shuffled unpredictably
**Fairness**: No player has advantage from shuffle algorithm
**Auditability**: Shuffle can be verified/replayed for disputes
**Consistency**: Same shuffle seed produces same card order (for testing and replay)

**Challenges**:
- Online games: Server controls shuffle (client cannot verify in real-time)
- Offline games: Client controls shuffle (no server validation)
- Sync: Offline shuffles must be accepted by server without recomputation
- Cross-platform: Go and Java must produce identical shuffle results

**Attack Vectors**:
- **Malicious server**: Predictable shuffles favoring house/bots
- **Malicious client**: Cherry-picking favorable offline shuffles
- **Replay attacks**: Reusing known shuffles
- **RNG bias**: Non-uniform distribution favoring certain cards

---

## Decision

**Server controls shuffle seed for online games; clients use verifiable deterministic shuffles.**

### Shuffle Strategy

#### Online Games (v1.0)

1. **Server-seeded shuffle**:
   - Server generates cryptographically secure random seed (32 bytes)
   - Server performs Fisher-Yates shuffle with seed
   - Server includes seed in `DeckShuffled` event
   - Clients can verify shuffle by replaying with same seed

2. **Event format**:
```json
{
  "type": "DeckShuffled",
  "game_id": "...",
  "seed": "base64(32_random_bytes)",
  "timestamp": "2025-10-02T01:30:00Z",
  "algorithm": "fisher-yates-v1"
}
```

#### Offline Games (v1.0)

1. **Client-seeded shuffle**:
   - Client generates crypto-random seed
   - Client stores seed in event log
   - On sync: Server accepts client's shuffle (trusts signature)
   - Future: Server can replay shuffle to verify fairness (v1.1)

2. **Same event format** (ensures consistency)

#### Provable Fairness (v2.0 - Future)

Commit-reveal protocol:
1. Server commits to shuffle: `commit = hash(seed + deck)`
2. Client contributes entropy: `client_random`
3. Final seed: `sha256(server_seed + client_random)`
4. Server reveals `server_seed`, client verifies commit
5. Both parties can verify final shuffle

### Shuffle Algorithm

**Fisher-Yates (Knuth) Shuffle**:
```go
func ShuffleDeck(seed []byte) []Card {
    rng := NewSeededRNG(seed) // Deterministic RNG from seed
    deck := NewDeck() // [A♠, 2♠, ..., K♣]

    for i := len(deck) - 1; i > 0; i-- {
        j := rng.Intn(i + 1) // 0 <= j <= i
        deck[i], deck[j] = deck[j], deck[i]
    }

    return deck
}
```

**Requirements**:
- **Deterministic**: Same seed → same deck order (every time)
- **Uniform**: Each permutation equally likely (~1/52! probability)
- **Cross-platform**: Go and Java produce identical results for same seed

### Seeded RNG Implementation

```go
// Use ChaCha20 stream cipher as CSPRNG (crypto-secure, deterministic)
type SeededRNG struct {
    cipher cipher.Stream
}

func NewSeededRNG(seed []byte) *SeededRNG {
    key := sha256.Sum256(seed) // 32-byte key
    nonce := make([]byte, chacha20.NonceSize) // Zero nonce
    cipher, _ := chacha20.NewUnauthenticatedCipher(key[:], nonce)
    return &SeededRNG{cipher: cipher}
}

func (r *SeededRNG) Intn(n int) int {
    // Generate uniform random int in [0, n)
    bytes := make([]byte, 8)
    r.cipher.XORKeyStream(bytes, bytes)
    val := binary.BigEndian.Uint64(bytes)
    return int(val % uint64(n))
}
```

**Java equivalent**: Use `javax.crypto.Cipher` with ChaCha20

---

## Consequences

### Positive

- **Testable**: Deterministic shuffles enable repeatable tests
- **Auditable**: Seed in event log allows shuffle verification
- **Fair**: Crypto-random seed ensures unpredictability
- **Cross-platform**: Same algorithm works in Go and Java
- **Replay-friendly**: Historical games can be reconstructed exactly

### Negative

- **Trust requirement**: Players must trust server's RNG (v1.0)
- **Offline trust gap**: Offline shuffles not validated until v1.1
- **Complexity**: Deterministic RNG more complex than `math/rand`
- **Performance**: ChaCha20 slower than simple LCG (~10x)

### Neutral

- ChaCha20 overhead negligible (shuffle once per round, ~1ms)
- Provable fairness in v2.0 adds protocol complexity
- Seed storage adds ~32 bytes per game to event log

---

## Alternatives Considered

### Alternative 1: Client-Controlled Shuffle (Trustless)

**Description**: Client shuffles deck, server trusts result

**Pros**:
- No server trust required
- Client verifies own fairness

**Cons**:
- Malicious client can generate favorable shuffles offline
- No way to detect cheating
- Multiplayer games require consensus (complex)

**Why rejected**: Unacceptable for online games (enables cheating)

### Alternative 2: Hardware RNG (Dedicated Device)

**Description**: Use hardware RNG (e.g., /dev/random, TPM)

**Pros**:
- True randomness (not pseudo-random)
- High entropy

**Cons**:
- Not deterministic (can't replay shuffles)
- Not portable (hardware-dependent)
- Slower (entropy pool exhaustion)

**Why rejected**: Determinism required for testing and replay

### Alternative 3: Blockchain-Based Shuffle

**Description**: Use blockchain randomness beacon (Chainlink VRF, etc.)

**Pros**:
- Decentralized trust
- Verifiable randomness

**Cons**:
- Extreme complexity and cost
- Latency (block confirmation time)
- Overkill for v1.0 scope

**Why rejected**: Too complex for initial version, revisit for v3.0

### Alternative 4: Collaborative Shuffle (Mental Poker)

**Description**: Cryptographic protocol where all players contribute entropy

**Pros**:
- Provably fair (no single party controls shuffle)
- Academic research exists

**Cons**:
- Very complex implementation
- High latency (multiple rounds)
- Not suitable for AI players (need to participate)

**Why rejected**: Complexity outweighs benefit for v1.0

---

## Migration Path

### Phase 1: Deterministic RNG Implementation (Week 1-2)

**Go (Client)**:
```go
// internal/domain/card/rng.go
import "golang.org/x/crypto/chacha20"

type DeterministicRNG struct {
    cipher cipher.Stream
    seed   []byte
}

func NewDeterministicRNG(seed []byte) *DeterministicRNG {
    // Implement ChaCha20-based RNG
}
```

**Java (Server)**:
```java
// core/domain/card/DeterministicRNG.java
import javax.crypto.Cipher;

public class DeterministicRNG {
    private Cipher cipher;
    private byte[] seed;

    public DeterministicRNG(byte[] seed) {
        // Implement ChaCha20-based RNG
    }

    public int nextInt(int bound) {
        // Generate uniform random int
    }
}
```

### Phase 2: Fisher-Yates Shuffle (Week 2)

```go
func (d *Deck) Shuffle(rng *DeterministicRNG) {
    for i := len(d.cards) - 1; i > 0; i-- {
        j := rng.Intn(i + 1)
        d.cards[i], d.cards[j] = d.cards[j], d.cards[i]
    }
}
```

### Phase 3: Golden Test Vectors (Week 3)

```json
// tests/golden/shuffle_vectors.json
{
  "vectors": [
    {
      "seed": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
      "expected_deck": [
        "3C", "10D", "KS", "2H", ...
      ],
      "description": "Zero seed"
    },
    {
      "seed": "//////////////////////////////////////////8=",
      "expected_deck": [
        "7H", "QC", "AS", ...
      ],
      "description": "Max seed"
    }
  ]
}
```

**Test**:
```go
func TestShuffleParity(t *testing.T) {
    vectors := loadGoldenVectors("shuffle_vectors.json")
    for _, v := range vectors {
        deck := NewDeck()
        rng := NewDeterministicRNG(v.Seed)
        deck.Shuffle(rng)
        assert.Equal(t, v.ExpectedDeck, deck.ToStrings())
    }
}
```

### Phase 4: Integration (Week 4)

**Server**:
```java
public GameEvent shuffleDeck(GameId gameId) {
    byte[] seed = SecureRandom.getInstanceStrong().generateSeed(32);
    Deck deck = new Deck();
    deck.shuffle(new DeterministicRNG(seed));

    return new DeckShuffled(gameId, Base64.encode(seed), Instant.now());
}
```

**Client** (on receiving DeckShuffled event):
```go
func (g *Game) HandleDeckShuffled(event *DeckShuffled) {
    seed, _ := base64.StdEncoding.DecodeString(event.Seed)
    rng := NewDeterministicRNG(seed)
    g.Deck.Shuffle(rng)
    // Verify shuffle if paranoid mode enabled
}
```

### Backward Compatibility

- v1.0: Simple server-seed shuffle
- v1.1: Add shuffle verification on sync
- v2.0: Provable fairness (commit-reveal)
- Algorithm versioning: `algorithm: "fisher-yates-v1"` allows future changes

### Rollback Plan

- If deterministic RNG proves problematic:
  - Fallback to `crypto/rand` for online games (non-deterministic, trust server)
  - Keep deterministic RNG only for testing
  - Exit criteria: ChaCha20 performance issues or platform incompatibility

---

## Related Decisions

- **ADR-001**: Event Sourcing - shuffle seed stored as event
- **ADR-002**: Server Authority - server controls shuffle for online games
- **ADR-003**: ed25519 Signatures - shuffle events are signed

---

## References

- [Fisher-Yates Shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)
- [ChaCha20 Stream Cipher](https://datatracker.ietf.org/doc/html/rfc8439)
- [Provably Fair Shuffling](https://en.wikipedia.org/wiki/Mental_poker)
- [How to Shuffle a Deck (Properly)](https://blog.codinghorror.com/the-danger-of-naivete/)

---

## Notes

### Fairness Testing

**Statistical Tests**:
```go
func TestShuffleUniformity(t *testing.T) {
    // Run 10,000 shuffles
    const trials = 10000
    cardPositions := make([][]int, 52)

    for i := 0; i < trials; i++ {
        seed := randomSeed()
        deck := NewDeck()
        deck.Shuffle(NewDeterministicRNG(seed))

        for pos, card := range deck.Cards() {
            cardPositions[card.Index()] = append(cardPositions[card.Index()], pos)
        }
    }

    // Chi-square test: each card should be evenly distributed across positions
    for cardIdx, positions := range cardPositions {
        chiSquare := calculateChiSquare(positions, 52)
        assert.Less(t, chiSquare, criticalValue(0.05),
            "Card %d distribution not uniform", cardIdx)
    }
}
```

### Performance Benchmarks

```
Shuffle (ChaCha20):  ~1ms per deck
Shuffle (crypto/rand): ~0.5ms per deck
Shuffle (math/rand):   ~0.1ms per deck
```

**Trade-off**: ChaCha20 is 10x slower than math/rand but provides:
- Crypto-security (unpredictable)
- Determinism (reproducible)
- Cross-platform consistency

**1ms overhead per round is acceptable.**

### Provable Fairness (v2.0 Preview)

```
Client-Server Commit-Reveal Protocol:

1. [Server] Generate server_seed (secret)
2. [Server] commit = SHA256(server_seed + deck_order)
3. [Server → Client] Send commit
4. [Client] Generate client_seed, send to server
5. [Server → Client] Reveal server_seed
6. [Client] Verify: SHA256(server_seed + expected_deck) == commit
7. [Both] final_seed = SHA256(server_seed + client_seed)
8. [Both] Shuffle deck with final_seed
9. [Both] Verify decks match
```

Benefits:
- Client can verify server didn't cheat
- Server can't predict client's contribution
- Transparent and auditable

Drawbacks:
- 2 extra round-trips (latency)
- More complex protocol
- Still requires trust in shuffle algorithm itself

### Seed Generation

**Server (Java)**:
```java
SecureRandom sr = SecureRandom.getInstanceStrong();
byte[] seed = sr.generateSeed(32); // 256 bits of entropy
```

**Client (Go)**:
```go
seed := make([]byte, 32)
if _, err := rand.Read(seed); err != nil {
    panic("RNG failure") // Catastrophic, should never happen
}
```

### Cross-Platform Consistency Verification

Weekly CI job:
```bash
# Run shuffle test on both platforms
go test -run TestShuffleParity > go_results.txt
./gradlew test --tests ShuffleParityTest > java_results.txt

# Compare outputs (must be identical)
diff go_results.txt java_results.txt || exit 1
```

### Future: Shuffle Auditing API

```
GET /api/games/{game_id}/shuffle
{
  "game_id": "...",
  "seed": "base64...",
  "algorithm": "fisher-yates-v1",
  "deck_order": ["3C", "10D", ...],
  "timestamp": "2025-10-02T01:30:00Z",
  "verifiable": true
}

POST /api/verify-shuffle
{
  "seed": "base64...",
  "algorithm": "fisher-yates-v1",
  "claimed_deck": ["3C", ...]
}
Response:
{
  "valid": true,
  "actual_deck": ["3C", ...],
  "matches": true
}
```
