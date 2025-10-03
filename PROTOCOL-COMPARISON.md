# Server-Client Protocol Comparison

**Generated**: 2025-10-04
**Purpose**: Step 7 - Client Synchronization Analysis

---

## 📊 Message Type Comparison

### Server → Client Message Types

| # | Server (Java) | Client (Go) | Status |
|---|---------------|-------------|--------|
| 1 | `REGISTER_SUCCESS` | `ServerRegisterSuccess` | ✅ Match |
| 2 | `REGISTER_FAILURE` | `ServerRegisterFailure` | ✅ Match |
| 3 | `MATCHING_STARTED` | `ServerMatchingStarted` | ✅ Match |
| 4 | `MATCHING_PROGRESS` | `ServerMatchingProgress` | ✅ Match |
| 5 | `MATCHING_COMPLETED` | `ServerMatchingCompleted` | ✅ Match |
| 6 | `MATCHING_CANCELLED` | `ServerMatchingCancelled` | ✅ Match |
| 7 | `GAME_STARTED` | `ServerGameStarted` | ✅ Match |
| 8 | `GAME_STATE_UPDATE` | `ServerGameStateUpdate` | ✅ Match |
| 9 | `PLAYER_ACTION` | `ServerPlayerAction` | ✅ Match |
| 10 | `TURN_CHANGED` | ❌ **Missing** | ⚠️ **ADD** |
| 11 | `ROUND_PROGRESSED` | ❌ **Missing** | ⚠️ **ADD** |
| 12 | `ROUND_COMPLETED` | `ServerRoundCompleted` | ✅ Match |
| 13 | `GAME_ENDED` | `ServerGameEnded` | ✅ Match |
| 14 | `CHAT_MESSAGE` | ❌ **Missing** | ⚠️ **ADD** |
| 15 | `ERROR` | `ServerError` | ✅ Match |
| 16 | `INVALID_ACTION` | `ServerInvalidAction` | ✅ Match |

**Summary**: 13/16 match (81%)
**Action Required**: Add 3 missing message types to Go client

---

## 🔧 GameState Payload Structure Comparison

### Server (Java) - `GameRoom.getGameStateMap()`

**Location**: `/pokerhole-server/src/main/java/dev/xiyo/pokerhole/server/room/GameRoom.java:288-330`

```json
{
  "roomId": "string",
  "roomName": "string",
  "playerCount": 2,
  "round": "PRE_FLOP",
  "pot": 1000,
  "currentBet": 200,
  "communityCards": [
    {"suit": "HEARTS", "rank": "ACE"},
    {"suit": "DIAMONDS", "rank": "KING"}
  ],
  "currentTurnPlayer": "Alice",
  "players": [
    {
      "nickname": "Alice",
      "chips": 8000,
      "status": "ACTIVE",
      "currentBet": 200
    }
  ]
}
```

### Client (Go) - `state.GameState`

**Location**: `/pokerhole-cli/internal/state/game_state.go:9-40`

```go
type GameState struct {
    GameID         string      // expects "gameId"
    Round          string      // expects "round"
    Pot            int         // expects "pot"
    CurrentBet     int         // expects "currentBet"
    CommunityCards []string    // expects []string
    Players        []PlayerState
    CurrentPlayer  string      // expects "currentPlayer"
    ValidActions   []string    // expects "validActions"
}

type PlayerState struct {
    ID       string   // expects "id"
    Nickname string   // expects "nickname"
    Chips    int      // expects "chips"
    Bet      int      // expects "bet"
    Status   string   // expects "status"
    Position int      // expects "position"
}
```

---

## ⚠️ Protocol Mismatches

### Critical Issues (Breaking)

#### 1. **Field Name Mismatch: `currentTurnPlayer` vs `currentPlayer`**

- **Server sends**: `currentTurnPlayer` (String, nickname)
- **Client expects**: `currentPlayer` (String)
- **Impact**: Client won't receive current turn information
- **Fix**: Change server to send `currentPlayer` OR change client to read `currentTurnPlayer`
- **Recommended**: Change server to `currentPlayer` (simpler field name)

**File**: `GameRoom.java:313`
```java
// BEFORE
state.put("currentTurnPlayer", currentPlayer != null ? currentPlayer.getNickName() : null);

// AFTER
state.put("currentPlayer", currentPlayer != null ? currentPlayer.getNickName() : null);
```

#### 2. **Community Cards Format Mismatch**

- **Server sends**: `List<Map<String, String>>` with `{"suit": "HEARTS", "rank": "ACE"}`
- **Client expects**: `[]string` (e.g., `["AH", "KD"]`)
- **Impact**: Client cannot parse community cards
- **Fix**: Server should send card strings like "AH" (Ace of Hearts) OR client should parse nested objects
- **Recommended**: Change server to send card strings (more compact)

**File**: `GameRoom.java:303-309`
```java
// BEFORE
List<Map<String, String>> communityCardsList = dealer.getCommunityCards().stream()
    .map(card -> Map.of(
        "suit", card.getSuit().name(),
        "rank", card.getRank().name()
    ))
    .collect(Collectors.toList());

// AFTER
List<String> communityCardsList = dealer.getCommunityCards().stream()
    .map(Card::toString)  // Assuming Card has toString() like "AS", "KH"
    .collect(Collectors.toList());
```

#### 3. **Field Name Mismatch: `roomId` vs `gameId`**

- **Server sends**: `roomId`
- **Client expects**: `gameId`
- **Impact**: Client won't receive game ID
- **Fix**: Change server to send `gameId` (since this is Texas Hold'em game context, not just a room)
- **Recommended**: Change server to `gameId`

**File**: `GameRoom.java:293`
```java
// BEFORE
state.put("roomId", id);

// AFTER
state.put("gameId", id);
```

#### 4. **Player Field Name Mismatch: `currentBet` vs `bet`**

- **Server sends**: `currentBet` (in player object)
- **Client expects**: `bet`
- **Impact**: Client won't display player bet amounts
- **Fix**: Change server to send `bet` OR change client to read `currentBet`
- **Recommended**: Change server to `bet` (simpler)

**File**: `GameRoom.java:322`
```java
// BEFORE
playerInfo.put("currentBet", p.player().getCurrentBet());

// AFTER
playerInfo.put("bet", p.player().getCurrentBet());
```

### Minor Issues (Non-Breaking)

#### 5. **Missing Server Fields**

Client expects but server doesn't send:
- `validActions` ([]string) - Which actions are valid for current player
- `id` (string) - Player ID in player object
- `position` (int) - Player position in player object

**Recommended**: Add these fields to server response

#### 6. **Unused Server Fields**

Server sends but client doesn't use:
- `roomName` (string) - Could be useful for display
- `playerCount` (int) - Could be useful for UI

**Recommended**: Client should add these fields to GameState struct

---

## 🎯 Required Changes Summary

### Server Changes (Priority: High)

**File**: `/pokerhole-server/src/main/java/dev/xiyo/pokerhole/server/room/GameRoom.java`

1. ✅ **Line 293**: Change `roomId` → `gameId`
2. ✅ **Line 303-309**: Change community cards format to strings
3. ✅ **Line 313**: Change `currentTurnPlayer` → `currentPlayer`
4. ✅ **Line 322**: Change `currentBet` → `bet` (in player object)
5. ⚠️ **Add**: `validActions` field (list of valid actions for current player)
6. ⚠️ **Add**: `id` field in player object (player UUID)
7. ⚠️ **Add**: `position` field in player object

### Client Changes (Priority: High)

**File**: `/pokerhole-cli/internal/network/client.go`

1. ✅ **Add**: `ServerTurnChanged` message type
2. ✅ **Add**: `ServerRoundProgressed` message type
3. ✅ **Add**: `ServerChatMessage` message type

**File**: `/pokerhole-cli/internal/state/game_state.go`

4. ⚠️ **Add**: `RoomName` field (optional, for display)
5. ⚠️ **Add**: `PlayerCount` field (optional, for UI)

---

## 📝 Implementation Order

### Phase 1: Critical Fixes (Breaks protocol)

1. **Server**: Fix 4 field name mismatches
2. **Client**: Add 3 missing message types
3. **Test**: Verify basic game state updates work

### Phase 2: Enhancements (Improves UX)

4. **Server**: Add `validActions`, `id`, `position` fields
5. **Client**: Add `RoomName`, `PlayerCount` fields
6. **Test**: Verify enhanced features work

---

## ✅ Verification Checklist

After changes:

- [ ] Client can receive `GAME_STATE_UPDATE` and parse all fields
- [ ] Community cards display correctly
- [ ] Current turn player is highlighted correctly
- [ ] Player bet amounts display correctly
- [ ] Client handles `TURN_CHANGED` message
- [ ] Client handles `ROUND_PROGRESSED` message
- [ ] Client handles `CHAT_MESSAGE` message
- [ ] Integration test: Start game → Action → Round progression → Showdown

---

## 🔄 Next Steps

1. Apply server changes to `GameRoom.java`
2. Apply client changes to `client.go` and `game_state.go`
3. Run integration test
4. Update NEXT-STEP.md with completion status
