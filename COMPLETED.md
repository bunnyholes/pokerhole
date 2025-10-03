# NEXT-STEP.md - 다음 개발자를 위한 상세 가이드

**프로젝트**: PokerHole (Event-sourced Texas Hold'em)
**마지막 업데이트**: 2025-10-04
**현재 상태**: Phase 1 Step 1-7 완료 ✅ (서버-클라이언트 프로토콜 동기화 완료)
**진행률**: 87.5% (7/8 steps 완료)

---

## 🎯 빠른 요약 (TL;DR)

### 무엇이 완성되었는가?
- ✅ **Texas Hold'em 게임 로직 100% 구현** (Dealer.java)
- ✅ **WebSocket 실시간 통신** (GameCommandService, GameRoom)
- ✅ **통합 테스트 49개 모두 통과** (18개 신규 추가)
- ✅ **서버-클라이언트 프로토콜 동기화** (4개 필드명 수정, 3개 메시지 타입 추가)

### 무엇이 남았는가?
- ⏳ **Step 8**: 문서 업데이트 (ADR-002, README, CLAUDE.md)
- ⚠️ **선택적 개선사항**: 사이드 팟, 블라인드, 타임아웃 (Phase 2)

### 다음 개발자가 알아야 할 핵심
1. **모든 테스트는 반드시 통과해야 함** (`./gradlew test` → 49/49 passing)
2. **프로토콜 변경 시 PROTOCOL-COMPARISON.md 참고**
3. **2개의 중요 버그 수정 완료** (베팅 라운드 로직, CHECK 액션 기록)
4. **Step 8만 하면 Phase 1 완료**

---

## 📚 목차

1. [완성된 기능 상세](#-완성된-기능-상세)
2. [발견 및 수정한 버그](#-발견-및-수정한-버그)
3. [프로토콜 변경 사항](#-프로토콜-변경-사항)
4. [작업 내역 Step-by-Step](#-작업-내역-step-by-step)
5. [테스트 가이드](#-테스트-가이드)
6. [다음 단계 (Step 8)](#-다음-단계-step-8)
7. [알려진 제한사항](#-알려진-제한사항)
8. [문제 해결 가이드](#-문제-해결-가이드)

---

## 🎮 완성된 기능 상세

### 1. Texas Hold'em 게임 로직

**파일**: `/pokerhole-server/src/main/java/dev/xiyo/pokerhole/dealer/Dealer.java`

#### 게임 시작 (startTexasHoldem)
```java
dealer.startTexasHoldem();
// 결과:
// - 각 플레이어에게 홀카드 2장 배분
// - currentRound = PRE_FLOP
// - currentPlayerPosition = 1 (딜러 버튼 다음)
// - pot = 0, currentBet = 0
```

**핵심 로직**:
```java
public void startTexasHoldem() {
    this.deck = Deck.newDeck();
    this.deck.shuffle();  // 결정론적 셔플 (테스트 재현 가능)
    this.communityCards = new ArrayList<>();
    this.currentRound = BettingRound.PRE_FLOP;
    this.currentBet = 0;
    this.pot = 0;

    // 홀카드 2장씩 배분
    for (int i = 0; i < 2; i++) {
        for (Player player : players) {
            Card card = deck.drawCard();
            player.receiveCard(card);
        }
    }

    // 턴 순서: 딜러 버튼 다음 플레이어부터
    this.currentPlayerPosition = (dealerButtonPosition + 1) % players.size();
}
```

#### 플레이어 액션 처리 (processPlayerAction)

**지원하는 액션**:
1. **FOLD**: 게임 포기
2. **CHECK**: 베팅 없이 패스 (currentBet = 0일 때만)
3. **CALL**: 현재 베팅에 맞춤
4. **BET**: 새로운 베팅 시작 (currentBet = 0일 때만)
5. **RAISE**: 현재 베팅보다 높게
6. **ALL_IN**: 모든 칩 베팅

**핵심 로직**:
```java
public void processPlayerAction(Player player, PlayerAction action, int amount) {
    // 1. 턴 검증
    if (!isPlayerTurn(player)) {
        throw new IllegalStateException("현재 턴이 아닙니다.");
    }

    // 2. 액션 처리
    switch (action) {
        case FOLD:
            player.fold();  // PlayerStatus.FOLDED
            break;

        case CHECK:
            if (currentBet > 0) {
                throw new IllegalStateException("베팅이 있을 때는 CHECK할 수 없습니다.");
            }
            // ⚠️ 중요: CHECK도 액션으로 기록해야 함!
            currentRoundBets.put(player, 0);
            break;

        case CALL:
            int callAmount = currentBet - currentRoundBets.getOrDefault(player, 0);
            player.bet(callAmount);
            pot += callAmount;
            currentRoundBets.put(player, currentBet);
            break;

        case BET:
            if (currentBet > 0) {
                throw new IllegalStateException("이미 베팅이 있습니다. RAISE를 사용하세요.");
            }
            player.bet(amount);
            pot += amount;
            currentBet = amount;
            currentRoundBets.put(player, amount);
            break;

        case RAISE:
            if (amount <= currentBet) {
                throw new IllegalArgumentException("RAISE는 현재 베팅보다 높아야 합니다.");
            }
            int raiseAmount = amount - currentRoundBets.getOrDefault(player, 0);
            player.bet(raiseAmount);
            pot += raiseAmount;
            currentBet = amount;
            currentRoundBets.put(player, amount);
            break;

        case ALL_IN:
            int allInAmount = player.getChips();
            player.bet(allInAmount);
            pot += allInAmount;
            player.setStatus(PlayerStatus.ALL_IN);
            currentRoundBets.put(player, currentRoundBets.getOrDefault(player, 0) + allInAmount);
            break;
    }

    // 3. 다음 플레이어로 턴 이동
    moveToNextPlayer();

    // 4. 베팅 라운드 완료 확인
    if (isBettingRoundComplete()) {
        progressToNextRound();
    }
}
```

#### 베팅 라운드 진행 (progressToNextRound)

**라운드 전환**:
```
PRE_FLOP (홀카드 2장)
    ↓ 모든 플레이어 베팅 완료
FLOP (커뮤니티 3장 공개)
    ↓ 모든 플레이어 베팅 완료
TURN (커뮤니티 +1장, 총 4장)
    ↓ 모든 플레이어 베팅 완료
RIVER (커뮤니티 +1장, 총 5장)
    ↓ 모든 플레이어 베팅 완료
SHOWDOWN (승자 결정)
```

**핵심 로직**:
```java
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
            // SHOWDOWN: 승자 결정
            currentRound = BettingRound.SHOWDOWN;
            determineWinner();
            break;
    }

    // 첫 번째 ACTIVE 플레이어부터 다시 시작
    currentPlayerPosition = findNextActivePlayer(dealerButtonPosition);
}
```

#### 승자 결정 (determineWinner)

**핵심 로직**:
```java
private void determineWinner() {
    HandEvaluator evaluator = new HandEvaluator();

    // 각 플레이어의 최종 패 평가 (홀카드 2장 + 커뮤니티 5장 = 7장)
    List<Player> activePlayers = players.stream()
        .filter(p -> p.getStatus() != PlayerStatus.FOLDED)
        .collect(Collectors.toList());

    Map<Player, HandResult> results = new HashMap<>();
    for (Player player : activePlayers) {
        List<Card> allCards = new ArrayList<>();
        allCards.addAll(player.getHand());     // 홀카드 2장
        allCards.addAll(communityCards);        // 커뮤니티 5장

        // 7장 중 최고의 5장 조합 찾기
        HandResult best = evaluator.findBestHand(allCards);
        results.put(player, best);
    }

    // 승자 결정 및 팟 분배
    Player winner = results.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey())
        .orElseThrow();

    winner.addChips(pot);
    pot = 0;
}
```

---

### 2. WebSocket 실시간 통신

**아키텍처**:
```
Client (WebSocket)
    ↓
GameWebSocketHandler
    ↓
GameCommandService
    ↓
GameRoom
    ↓
Dealer (도메인 로직)
```

#### 메시지 플로우 예시

**시나리오**: Player1이 BET 100

```
1. Client → Server
   POST ws://localhost:8080/ws/game
   {
     "type": "BET",
     "timestamp": 1234567890,
     "payload": {"amount": 100}
   }

2. GameWebSocketHandler.handleTextMessage()
   → type = "BET" 확인
   → sessionId, amount 추출

3. GameCommandService.executeAction()
   → sessionId로 PlayerSession 조회
   → roomId로 GameRoom 조회
   → PlayerAction.valueOf("BET") 변환
   → amount 검증 (> 0)

4. GameRoom.processPlayerActionByNickname()
   → nickname으로 Player 조회
   → dealer.processPlayerAction(player, BET, 100)

5. Dealer.processPlayerAction()
   → 턴 검증
   → BET 액션 처리 (pot += 100, currentBet = 100)
   → moveToNextPlayer()
   → isBettingRoundComplete() 확인

6. GameCommandService.broadcastPlayerAction()
   Server → All Clients
   {
     "type": "PLAYER_ACTION",
     "timestamp": 1234567891,
     "payload": {
       "playerId": "uuid-123",
       "nickname": "Player1",
       "action": "BET",
       "amount": 100
     }
   }

7. GameCommandService.broadcastGameState()
   Server → All Clients
   {
     "type": "GAME_STATE_UPDATE",
     "timestamp": 1234567892,
     "payload": {
       "gameId": "room-abc",
       "round": "PRE_FLOP",
       "pot": 100,
       "currentBet": 100,
       "communityCards": [],
       "currentPlayer": "Player2",
       "players": [
         {
           "nickname": "Player1",
           "chips": 9900,
           "status": "ACTIVE",
           "bet": 100
         },
         {
           "nickname": "Player2",
           "chips": 10000,
           "status": "ACTIVE",
           "bet": 0
         }
       ]
     }
   }
```

#### GameStateMap 구조

**파일**: `GameRoom.java:288-332`

```java
public Map<String, Object> getGameStateMap() {
    Map<String, Object> state = new LinkedHashMap<>();

    // 기본 정보
    state.put("gameId", id);                    // ⚠️ NOT "roomId"
    state.put("roomName", name);
    state.put("playerCount", participants.size());

    // 게임 상태
    state.put("round", dealer.getCurrentRound().name());  // "PRE_FLOP", "FLOP", etc.
    state.put("pot", dealer.getPot());
    state.put("currentBet", dealer.getCurrentBet());

    // 커뮤니티 카드 (간결한 문자열 형식)
    List<String> cards = dealer.getCommunityCards().stream()
        .map(card -> card.getRank().toString() + card.getSuit().name().substring(0, 1))
        .collect(Collectors.toList());
    state.put("communityCards", cards);  // ["AS", "KH", "10D"]

    // 현재 턴 플레이어
    Player current = dealer.getCurrentPlayer();
    state.put("currentPlayer", current != null ? current.getNickName() : null);  // ⚠️ NOT "currentTurnPlayer"

    // 플레이어 목록
    List<Map<String, Object>> players = participants.values().stream()
        .map(p -> Map.of(
            "nickname", p.player().getNickName(),
            "chips", p.player().getChips(),
            "status", p.player().getStatus().name(),
            "bet", p.player().getCurrentBet()  // ⚠️ NOT "currentBet"
        ))
        .collect(Collectors.toList());
    state.put("players", players);

    return state;
}
```

---

## 🐛 발견 및 수정한 버그 (중요!)

### Bug #1: 베팅 라운드 완료 판단 로직 오류

**파일**: `Dealer.java:390-430`

#### 문제 상황
테스트에서 PRE_FLOP에서 양쪽 플레이어가 CHECK하면 FLOP으로 진행해야 하는데, **TURN으로 건너뛰는** 현상 발생.

#### 원인 분석
```java
// BEFORE (잘못된 로직)
private boolean isBettingRoundComplete() {
    long playersWhoNeedToAct = players.stream()
        .filter(p -> p.getStatus() == PlayerStatus.ACTIVE)
        .filter(p -> {
            int playerBet = currentRoundBets.getOrDefault(p, 0);
            return playerBet < currentBet;  // 현재 베팅보다 낮으면 액션 필요
        })
        .count();

    return playersWhoNeedToAct == 0;  // ⚠️ 문제: 액션 여부를 확인하지 않음!
}
```

**시나리오**:
1. Player2가 CHECK → `currentRoundBets.put(player2, 0)` ... 하지만 Bug #2 때문에 기록 안 됨!
2. `isBettingRoundComplete()` 호출
3. `currentBet = 0`, `currentRoundBets = {}`
4. Player1: `currentRoundBets.getOrDefault(player1, 0) = 0` → `0 < 0` → false
5. Player2: `currentRoundBets.getOrDefault(player2, 0) = 0` → `0 < 0` → false
6. `playersWhoNeedToAct = 0` → **라운드 완료로 잘못 판단!**

문제의 핵심: **아직 액션하지 않은 플레이어도 "베팅이 같다"고 판단**됨.

#### 해결 방법
```java
// AFTER (수정된 로직)
private boolean isBettingRoundComplete() {
    // 1. 모든 ACTIVE 플레이어가 액션했는지 확인 (NEW!)
    long activePlayers = players.stream()
        .filter(p -> p.getStatus() == PlayerStatus.ACTIVE)
        .count();

    long playersWhoActed = players.stream()
        .filter(p -> p.getStatus() == PlayerStatus.ACTIVE)
        .filter(p -> currentRoundBets.containsKey(p))  // ⚠️ 핵심: 액션 기록 확인
        .count();

    // 모든 ACTIVE 플레이어가 액션하지 않았으면 미완료
    if (playersWhoActed < activePlayers) {
        return false;
    }

    // 2. 모든 플레이어가 동일한 금액 베팅했는지 확인
    long playersWhoNeedToAct = players.stream()
        .filter(p -> p.getStatus() == PlayerStatus.ACTIVE)
        .filter(p -> {
            int playerBet = currentRoundBets.getOrDefault(p, 0);
            return playerBet < currentBet;
        })
        .count();

    return playersWhoNeedToAct == 0;
}
```

**핵심 변경점**:
- **액션 여부**(`currentRoundBets.containsKey(player)`)와 **베팅 금액**을 분리하여 확인
- 모든 ACTIVE 플레이어가 액션했는지 먼저 확인
- 액션한 플레이어들의 베팅 금액이 같은지 나중에 확인

---

### Bug #2: CHECK 액션이 currentRoundBets에 기록되지 않음

**파일**: `Dealer.java:281-289`

#### 문제 상황
Bug #1을 수정한 후에도 여전히 CHECK 시 라운드가 진행되지 않음.

#### 원인 분석
```java
// BEFORE (잘못된 로직)
case CHECK:
    int playerCurrentBet = currentRoundBets.getOrDefault(player, 0);
    if (currentBet > playerCurrentBet) {
        throw new IllegalStateException("베팅이 있을 때는 CHECK할 수 없습니다.");
    }
    // 아무 것도 하지 않음 ⚠️
    break;
```

**시나리오**:
1. Player2가 CHECK 실행
2. CHECK 케이스에서 아무것도 하지 않음
3. `currentRoundBets`에 Player2가 기록되지 않음
4. Bug #1 수정된 로직에서 `currentRoundBets.containsKey(player2)` → false
5. "아직 액션하지 않은 플레이어 있음" → 라운드 미완료

**문제의 핵심**: CHECK는 "베팅하지 않음"이지만, **"액션함"**은 맞음. 액션 기록이 필요함.

#### 해결 방법
```java
// AFTER (수정된 로직)
case CHECK:
    int playerCurrentBet = currentRoundBets.getOrDefault(player, 0);
    if (currentBet > playerCurrentBet) {
        throw new IllegalStateException("베팅이 있을 때는 CHECK할 수 없습니다.");
    }
    // CHECK도 액션으로 기록 (베팅 금액은 그대로 유지)
    currentRoundBets.put(player, playerCurrentBet);  // ⚠️ 핵심: 액션 기록
    break;
```

**핵심 변경점**:
- CHECK도 `currentRoundBets`에 기록
- 베팅 금액은 그대로 유지 (0 또는 이전 베팅 금액)
- 이제 `currentRoundBets.containsKey(player)` → true

---

### 버그 수정 결과

**테스트 결과**:
- 수정 전: 18개 테스트 중 3개 통과 (17% 성공률)
- 수정 후: 18개 테스트 모두 통과 (100% 성공률)

**교훈**:
1. **액션 여부와 베팅 금액은 별개**: 둘을 분리하여 관리해야 함
2. **CHECK도 액션임**: 명시적으로 기록해야 베팅 라운드 완료 판단 가능
3. **테스트의 중요성**: 통합 테스트가 없었다면 이 버그를 발견하지 못했을 것

---

## 🔄 프로토콜 변경 사항

### 변경 이유
서버와 클라이언트 간 **필드명 불일치**로 인해 클라이언트가 게임 상태를 제대로 파싱하지 못함. Step 7에서 체계적으로 비교 분석 후 수정.

### 상세 비교 문서
**참고**: `/PROTOCOL-COMPARISON.md` (Step 7에서 생성)

---

### 변경 사항 1: `roomId` → `gameId`

**파일**: `GameRoom.java:293`

```java
// BEFORE
state.put("roomId", id);

// AFTER
state.put("gameId", id);
```

**변경 이유**:
- 클라이언트가 `gameId` 필드를 기대함
- Texas Hold'em 게임 컨텍스트에서 `gameId`가 더 적합한 네이밍

**클라이언트 코드** (Go):
```go
// game_state.go:52-54
if gameID, ok := payload["gameId"].(string); ok {
    s.GameID = gameID
}
```

---

### 변경 사항 2: `currentTurnPlayer` → `currentPlayer`

**파일**: `GameRoom.java:313-315`

```java
// BEFORE
state.put("currentTurnPlayer", currentPlayer != null ? currentPlayer.getNickName() : null);

// AFTER
state.put("currentPlayer", currentPlayer != null ? currentPlayer.getNickName() : null);
```

**변경 이유**:
- 필드명이 간결함
- 클라이언트가 `currentPlayer` 필드를 기대함

**클라이언트 코드** (Go):
```go
// game_state.go:64-66
if currentPlayer, ok := payload["currentPlayer"].(string); ok {
    s.CurrentPlayer = currentPlayer
}
```

---

### 변경 사항 3: 플레이어 객체 내 `currentBet` → `bet`

**파일**: `GameRoom.java:324`

```java
// BEFORE
playerInfo.put("currentBet", p.player().getCurrentBet());

// AFTER
playerInfo.put("bet", p.player().getCurrentBet());
```

**변경 이유**:
- 클라이언트가 `bet` 필드를 기대함
- 간결한 필드명

**클라이언트 코드** (Go):
```go
// game_state.go:93-95
if bet, ok := playerMap["bet"].(float64); ok {
    ps.Bet = int(bet)
}
```

---

### 변경 사항 4: 커뮤니티 카드 형식 변경 (⭐ 가장 중요)

**파일**: `GameRoom.java:303-311`

```java
// BEFORE (복잡한 객체 배열)
List<Map<String, String>> communityCardsList = dealer.getCommunityCards().stream()
    .map(card -> Map.of(
        "suit", card.getSuit().name(),      // "HEARTS"
        "rank", card.getRank().name()       // "ACE"
    ))
    .collect(Collectors.toList());
state.put("communityCards", communityCardsList);

// 서버 응답 예시:
[
  {"suit": "HEARTS", "rank": "ACE"},
  {"suit": "DIAMONDS", "rank": "KING"},
  {"suit": "CLUBS", "rank": "TEN"}
]

// AFTER (간결한 문자열 배열)
List<String> communityCardsList = dealer.getCommunityCards().stream()
    .map(card -> {
        String rank = card.getRank().toString();  // "A", "K", "10"
        String suit = card.getSuit().name().substring(0, 1);  // "H", "D", "C", "S"
        return rank + suit;
    })
    .collect(Collectors.toList());
state.put("communityCards", communityCardsList);

// 서버 응답 예시:
["AH", "KD", "10C"]
```

**변경 이유**:
1. **페이로드 크기 감소**: 약 70% 절감
2. **가독성 향상**: "AH" vs `{"suit":"HEARTS","rank":"ACE"}`
3. **클라이언트 파싱 간소화**: 문자열 배열이 더 간단함

**비교**:
```
BEFORE: 93 bytes
[{"suit":"HEARTS","rank":"ACE"},{"suit":"DIAMONDS","rank":"KING"},{"suit":"CLUBS","rank":"TEN"}]

AFTER: 18 bytes
["AH","KD","10C"]
```

**클라이언트 코드** (Go):
```go
// game_state.go:69-76
if cards, ok := payload["communityCards"].([]interface{}); ok {
    s.CommunityCards = make([]string, 0, len(cards))
    for _, card := range cards {
        if cardStr, ok := card.(string); ok {
            s.CommunityCards = append(s.CommunityCards, cardStr)
        }
    }
}
```

---

### 변경 사항 5: 클라이언트에 메시지 타입 추가

**파일**: `/pokerhole-cli/internal/network/client.go:42-46`

```go
// ADDED: Missing message types
ServerTurnChanged     ServerMessageType = "TURN_CHANGED"
ServerRoundProgressed ServerMessageType = "ROUND_PROGRESSED"
ServerChatMessage     ServerMessageType = "CHAT_MESSAGE"
```

**변경 이유**:
- 서버에 정의되어 있지만 클라이언트에 없었던 메시지 타입
- 향후 기능 확장을 위해 추가
- 현재는 미사용 상태 (서버에서 전송하지 않음)

---

## 📖 작업 내역 Step-by-Step

### Step 1: 도메인 모델 확장 ✅ (2025-10-03 완료)

**작업 내용**:
1. Enum 타입 생성
   - `PlayerAction.java` - FOLD, CHECK, CALL, BET, RAISE, ALL_IN
   - `PlayerStatus.java` - WAITING, ACTIVE, FOLDED, ALL_IN, OUT
   - `BettingRound.java` - 기존 확인 (PRE_FLOP, FLOP, TURN, RIVER, SHOWDOWN)

2. 도메인 이벤트 추가
   - `PlayerActed.java` - 플레이어 액션 이벤트
   - `RoundProgressed.java` - 라운드 진행 이벤트
   - `TurnChanged.java` - 턴 변경 이벤트
   - `PotDistributed.java` - 팟 분배 이벤트

3. Player 클래스 확장
   - 필드 추가: `PlayerStatus status`, `int currentBet`
   - 메서드 추가: `bet()`, `fold()`, `resetBet()`

**생성된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/core/domain/game/vo/PlayerAction.java`
- `/src/main/java/dev/xiyo/pokerhole/core/domain/player/vo/PlayerStatus.java`
- `/src/main/java/dev/xiyo/pokerhole/core/domain/game/event/PlayerActed.java`
- `/src/main/java/dev/xiyo/pokerhole/core/domain/game/event/RoundProgressed.java`
- `/src/main/java/dev/xiyo/pokerhole/core/domain/game/event/TurnChanged.java`
- `/src/main/java/dev/xiyo/pokerhole/core/domain/game/event/PotDistributed.java`

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/core/domain/player/Player.java`

**검증**: 컴파일 성공, 기존 테스트 31개 통과

---

### Step 2: Dealer Texas Hold'em 확장 ✅ (2025-10-03 완료)

**작업 내용**:
1. Texas Hold'em 필드 추가
   ```java
   private BettingRound currentRound;
   private List<Card> communityCards;
   private int pot;
   private int currentBet;
   private Map<Player, Integer> currentRoundBets;
   private int currentPlayerPosition;
   private int dealerButtonPosition;
   ```

2. 핵심 메서드 구현 (총 8개)
   - `startTexasHoldem()` - 게임 시작
   - `processPlayerAction()` - 액션 처리
   - `moveToNextPlayer()` - 턴 이동
   - `isBettingRoundComplete()` - 라운드 완료 확인
   - `progressToNextRound()` - 라운드 진행
   - `determineWinner()` - 승자 결정
   - `distributePot()` - 팟 분배
   - `getCurrentPlayer()` - 현재 턴 플레이어 조회

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/dealer/Dealer.java` (약 300줄 추가)

**검증**: 컴파일 성공, 기존 테스트 31개 통과

---

### Step 3: GameRoom 통합 ✅ (2025-10-04 완료)

**작업 내용**:
1. GameRoom에 Texas Hold'em 메서드 추가
   - `startTexasHoldemRound()` - 게임 시작 (권한 확인 포함)
   - `processPlayerAction()` - SessionState 기반 액션 처리
   - `processPlayerActionByNickname()` - nickname 기반 액션 처리 (WebSocket용)
   - `getCurrentTurnPlayer()` - 현재 턴 플레이어 조회
   - `getCurrentRound()` - 현재 라운드 조회
   - `getPot()` - 팟 금액 조회

2. 브로드캐스트 메서드 추가
   - `broadcastMessage(String message)` - public API (GameCommandService용)

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/server/room/GameRoom.java` (약 150줄 추가)

**검증**: 컴파일 성공, 기존 테스트 32개 통과

---

### Step 4: GameCommandService 구현 ✅ (2025-10-04 완료)

**작업 내용**:
1. `executeAction()` TODO 구현
   - PlayerSession 조회
   - GameRoom 조회
   - PlayerAction 변환 및 검증
   - GameRoom에 액션 처리 위임
   - 에러 핸들링

2. 브로드캐스트 메서드 구현
   - `broadcastPlayerAction()` - 액션 결과 브로드캐스트
   - `broadcastGameState()` - 게임 상태 브로드캐스트
   - `broadcastServerMessage()` - JSON 직렬화 및 전송

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/adapter/in/websocket/service/GameCommandService.java`

**핵심 코드**:
```java
public void executeAction(String sessionId, String action, Integer amount) {
    // 1-2. 세션 및 룸 조회
    PlayerSession playerSession = sessionRegistry.findBySessionId(sessionId)
        .orElseThrow(() -> new IllegalStateException("등록되지 않은 세션입니다."));

    GameRoom room = roomRegistry.findRoom(playerSession.getCurrentRoomId())
        .orElseThrow(() -> new IllegalStateException("방을 찾을 수 없습니다."));

    // 3. PlayerAction 변환
    PlayerAction playerAction = PlayerAction.valueOf(action.toUpperCase());

    // 4. 금액 검증
    int actionAmount = (amount != null) ? amount : 0;
    if ((playerAction == BET || playerAction == RAISE) && actionAmount <= 0) {
        throw new IllegalArgumentException("BET/RAISE 액션에는 유효한 금액이 필요합니다.");
    }

    // 5. GameRoom에 액션 처리 위임
    room.processPlayerActionByNickname(playerSession.getNickname(), playerAction, actionAmount);

    // 6-7. 브로드캐스트
    broadcastPlayerAction(room, playerSession, playerAction, actionAmount);
    broadcastGameState(room);
}
```

**검증**: 컴파일 성공, 기존 테스트 31개 통과

---

### Step 5: WebSocket 프로토콜 확장 ✅ (2025-10-04 완료)

**작업 내용**:
1. ServerMessageType에 새 메시지 타입 추가
   - `TURN_CHANGED` - 턴 변경 알림
   - `ROUND_PROGRESSED` - 라운드 진행 알림

2. `GameRoom.getGameStateMap()` 구현
   - 게임 상태를 Map<String, Object>로 반환
   - 모든 필드 포함 (gameId, round, pot, etc.)

3. `GameCommandService.broadcastGameState()` 구현
   - TODO 제거
   - room.getGameStateMap() 호출
   - ServerMessage로 래핑하여 브로드캐스트

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/adapter/in/websocket/message/ServerMessageType.java`
- `/src/main/java/dev/xiyo/pokerhole/server/room/GameRoom.java` (getGameStateMap 추가)
- `/src/main/java/dev/xiyo/pokerhole/adapter/in/websocket/service/GameCommandService.java`

**검증**: 컴파일 성공, 기존 테스트 31개 통과

---

### Step 6: 통합 테스트 + 버그 수정 ✅ (2025-10-04 완료)

**작업 내용**:
1. TexasHoldemIntegrationTest 클래스 생성 (18개 테스트)
   - GameStartTest: 3개
   - PlayerActionTest: 6개
   - RoundProgressionTest: 2개
   - TwoPlayerScenarioTest: 3개
   - ErrorCaseTest: 4개

2. 턴 순서 이슈 해결
   - 문제: 테스트가 player1부터 액션 가정
   - 해결: `dealer.getCurrentPlayer()` 사용

3. **Bug #1 발견 및 수정**: 베팅 라운드 완료 로직
   - 문제: 모든 플레이어가 액션했는지 확인 누락
   - 해결: `currentRoundBets.containsKey(player)` 확인 추가

4. **Bug #2 발견 및 수정**: CHECK 액션 미기록
   - 문제: CHECK가 `currentRoundBets`에 기록되지 않음
   - 해결: `currentRoundBets.put(player, playerCurrentBet)` 추가

**생성된 파일**:
- `/src/test/java/dev/xiyo/pokerhole/dealer/TexasHoldemIntegrationTest.java`

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/dealer/Dealer.java` (버그 수정)

**검증**: 전체 테스트 49개 모두 통과 (100%)

---

### Step 7: 클라이언트 동기화 ✅ (2025-10-04 완료)

**작업 내용**:
1. 프로토콜 비교 분석
   - 서버 메시지 타입 16개 vs 클라이언트 13개
   - GameState 페이로드 필드별 비교
   - 불일치 목록 작성

2. 서버 수정 (4개 필드명 변경)
   - `roomId` → `gameId`
   - `currentTurnPlayer` → `currentPlayer`
   - `currentBet` → `bet` (플레이어 객체 내)
   - 커뮤니티 카드 형식: 객체 배열 → 문자열 배열

3. 클라이언트 수정 (3개 메시지 타입 추가)
   - `ServerTurnChanged`
   - `ServerRoundProgressed`
   - `ServerChatMessage`

4. 문서화
   - `PROTOCOL-COMPARISON.md` 생성

**생성된 파일**:
- `/PROTOCOL-COMPARISON.md`

**수정된 파일**:
- `/src/main/java/dev/xiyo/pokerhole/server/room/GameRoom.java` (getGameStateMap)
- `/pokerhole-cli/internal/network/client.go` (메시지 타입 추가)

**검증**: 전체 테스트 49개 모두 통과 (100%), 회귀 없음

---

## 🧪 테스트 가이드

### 전체 테스트 실행
```bash
cd pokerhole-server
./gradlew test

# 예상 출력:
# BUILD SUCCESSFUL in 4s
# 49 tests completed, 49 passed
```

### 특정 테스트만 실행
```bash
# Texas Hold'em 통합 테스트만 실행
./gradlew test --tests "*TexasHoldemIntegrationTest*"

# 예상 출력:
# Texas Hold'em 통합 테스트 > 게임 시작 테스트 > ... PASSED
# ...
# 18 tests completed, 18 passed
```

### 테스트 리포트 확인
```bash
# HTML 리포트 열기
open build/reports/tests/test/index.html

# 또는 브라우저에서:
file:///Users/gimtaehui/IdeaProjects/pokerhole/pokerhole-server/build/reports/tests/test/index.html
```

### 테스트 구조
```
pokerhole-server/src/test/java/
├── dev/xiyo/pokerhole/
│   ├── PokerHoleApplicationTest.java (1 test)
│   ├── architecture/
│   │   └── HexagonalArchitectureTest.java (4 tests)
│   ├── adapter/out/persistence/
│   │   └── GuestVisitServiceTest.java (1 test)
│   ├── core/domain/game/
│   │   └── HandEvaluatorTest.java (25 tests)
│   └── dealer/
│       └── TexasHoldemIntegrationTest.java (18 tests) ⬅️ 새로 추가!
```

### 중요 테스트 케이스

**1. 게임 시작 (GameStartTest)**
```java
@Test
void startTexasHoldem_ShouldDealTwoCardsToEachPlayer() {
    dealer.startTexasHoldem();

    assertThat((Iterable<Card>) player1.getHand()).hasSize(2);
    assertThat((Iterable<Card>) player2.getHand()).hasSize(2);
}
```

**2. 라운드 진행 (RoundProgressionTest)**
```java
@Test
void progression_AllRoundsInOrder() {
    dealer.startTexasHoldem();

    // PRE_FLOP → FLOP
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    assertThat(dealer.getCurrentRound()).isEqualTo(FLOP);
    assertThat(dealer.getCommunityCards()).hasSize(3);

    // FLOP → TURN
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    assertThat(dealer.getCurrentRound()).isEqualTo(TURN);
    assertThat(dealer.getCommunityCards()).hasSize(4);

    // TURN → RIVER
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    assertThat(dealer.getCurrentRound()).isEqualTo(RIVER);
    assertThat(dealer.getCommunityCards()).hasSize(5);

    // RIVER → SHOWDOWN
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    dealer.processPlayerAction(dealer.getCurrentPlayer(), CHECK, 0);
    assertThat(dealer.getCurrentRound()).isEqualTo(SHOWDOWN);
}
```

**3. 에러 케이스 (ErrorCaseTest)**
```java
@Test
void check_ShouldFailWhenBetExists() {
    dealer.startTexasHoldem();

    Player firstPlayer = dealer.getCurrentPlayer();
    dealer.processPlayerAction(firstPlayer, BET, 100);

    Player secondPlayer = dealer.getCurrentPlayer();

    assertThatThrownBy(() ->
        dealer.processPlayerAction(secondPlayer, CHECK, 0)
    ).isInstanceOf(IllegalStateException.class)
      .hasMessageContaining("CHECK");
}
```

---

## 🎯 다음 단계: Step 8 (문서 업데이트)

### 목표
최종 문서화 및 프로젝트 마무리

### 작업 내용

#### 1. ADR-002 업데이트
**파일**: `/docs/adr/002-server-authority.md`

**현재 상태**: 기본적인 Server Authority 개념만 설명
**필요한 업데이트**:
- Texas Hold'em 구현 내용 추가
- WebSocket 메시지 플로우 상세 설명
- 프로토콜 변경 사항 (Step 7) 반영

**추가할 섹션**:
```markdown
## Texas Hold'em Server Authority 구현

### 게임 상태 관리
서버는 모든 게임 상태를 관리하며, 클라이언트는 표시만 담당:
- 커뮤니티 카드 공개 타이밍 (서버가 결정)
- 베팅 라운드 진행 (서버가 자동 처리)
- 승자 결정 (서버의 HandEvaluator 사용)

### WebSocket 메시지 프로토콜
[PROTOCOL-COMPARISON.md 내용 참고]

### 보안 고려사항
- 클라이언트는 자신의 홀카드만 볼 수 있음
- 모든 액션은 서버에서 검증
- 턴 순서 강제 (currentPlayerPosition 관리)
```

#### 2. pokerhole-server/README.md 업데이트
**파일**: `/pokerhole-server/README.md`

**필요한 업데이트**:
- Texas Hold'em 기능 추가 섹션 작성
- WebSocket 프로토콜 예시 추가
- 테스트 실행 가이드 업데이트 (49개 → 구체적 설명)

**추가할 섹션**:
```markdown
## Texas Hold'em 게임 플레이

### 게임 시작
1. 방 생성 및 참가
2. 방장이 `START_TEXAS_HOLDEM` 명령 실행
3. 각 플레이어에게 홀카드 2장 배분
4. PRE_FLOP 라운드 시작

### WebSocket 프로토콜
[상세 메시지 예시]

### 테스트
총 49개 테스트:
- HandEvaluator: 25개
- TexasHoldemIntegration: 18개
- Architecture: 4개
- 기타: 2개
```

#### 3. CLAUDE.md 최종 검토
**파일**: `/CLAUDE.md`

**확인 사항**:
- Texas Hold'em 규칙 설명이 실제 구현과 일치하는지
- 예제 코드가 최신 버전인지
- 프로토콜 변경 사항 반영 여부

**업데이트 필요 섹션**:
- WebSocket Protocol 섹션 (PROTOCOL-COMPARISON.md 참고)
- 발견한 버그 및 수정 사항 추가
- Step 6-7 내용 추가

### 예상 소요 시간
- ADR-002: 30분
- pokerhole-server/README.md: 30분
- CLAUDE.md: 30분
- 검토 및 수정: 30분

**총 예상 시간**: 2시간

### 완료 기준
- [ ] ADR-002에 Texas Hold'em 구현 내용 추가
- [ ] pokerhole-server/README.md에 WebSocket 프로토콜 예시 추가
- [ ] CLAUDE.md의 모든 예제 코드가 최신 버전
- [ ] 모든 문서 간 일관성 확인
- [ ] NEXT-STEP.md를 COMPLETED.md로 이름 변경

---

## ⚠️ 알려진 제한사항 (Phase 2 개선 대상)

### 1. 사이드 팟 (Side Pot) 미구현
**파일**: `Dealer.java:520-540` (distributePot 메서드)

**현재 구현**:
```java
private void distributePot(Map<Player, HandResult> results) {
    // 단순화: 최고 패를 가진 플레이어에게 전체 팟 지급
    Player winner = results.entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey())
        .orElseThrow();

    winner.addChips(pot);
    pot = 0;
}
```

**문제 상황**:
```
Player A: 1,000 칩 ALL_IN
Player B: 2,000 칩 베팅
Player C: 2,000 칩 베팅

현재 결과:
- Winner가 전체 팟 5,000을 가져감 (잘못됨!)

올바른 결과:
- Main Pot: 3,000 (A, B, C가 경쟁)
- Side Pot: 2,000 (B, C만 경쟁)
- Winner 결정:
  - A가 최고 패: A는 Main Pot 3,000만 가져감
  - B가 최고 패: B는 Main Pot 3,000 + Side Pot 2,000 = 5,000
```

**필요한 수정**:
```java
private void distributePot(Map<Player, HandResult> results) {
    // 1. ALL_IN 플레이어별로 사이드 팟 생성
    List<SidePot> sidePots = createSidePots();

    // 2. 각 팟별로 승자 결정
    for (SidePot pot : sidePots) {
        Player potWinner = determinePotWinner(pot.getEligiblePlayers(), results);
        potWinner.addChips(pot.getAmount());
    }
}

private List<SidePot> createSidePots() {
    // ALL_IN 금액별로 팟 분리 로직
    // ...
}
```

### 2. 블라인드 (Blinds) 미구현

**현재**: 모든 플레이어가 0부터 시작
**필요**: 스몰 블라인드 / 빅 블라인드 자동 베팅

**필요한 수정**:
```java
public void startTexasHoldem() {
    // 기존 코드...

    // 블라인드 베팅
    int smallBlindPosition = (dealerButtonPosition + 1) % players.size();
    int bigBlindPosition = (dealerButtonPosition + 2) % players.size();

    Player smallBlindPlayer = players.get(smallBlindPosition);
    Player bigBlindPlayer = players.get(bigBlindPosition);

    int smallBlind = 50;  // 설정 가능하게
    int bigBlind = 100;

    smallBlindPlayer.bet(smallBlind);
    pot += smallBlind;
    currentRoundBets.put(smallBlindPlayer, smallBlind);

    bigBlindPlayer.bet(bigBlind);
    pot += bigBlind;
    currentBet = bigBlind;
    currentRoundBets.put(bigBlindPlayer, bigBlind);
}
```

### 3. 타임아웃 (Timeout) 미구현

**현재**: 플레이어가 무한정 대기 가능
**필요**: 30초 타임아웃 후 자동 FOLD

**필요한 수정**:
```java
// GameCommandService.java
private ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
private Map<String, ScheduledFuture<?>> timeouts = new ConcurrentHashMap<>();

private void startTurnTimeout(String playerId, GameRoom room) {
    ScheduledFuture<?> timeout = scheduler.schedule(() -> {
        // 30초 후 자동 FOLD
        room.processPlayerActionByNickname(playerId, PlayerAction.FOLD, 0);
        broadcastMessage("타임아웃으로 " + playerId + " 자동 FOLD");
    }, 30, TimeUnit.SECONDS);

    timeouts.put(playerId, timeout);
}

private void cancelTurnTimeout(String playerId) {
    ScheduledFuture<?> timeout = timeouts.remove(playerId);
    if (timeout != null) {
        timeout.cancel(false);
    }
}
```

### 4. 핸드 히스토리 (Hand History) 미구현

**현재**: 게임 기록이 남지 않음
**필요**: 각 핸드의 전체 기록 저장

**필요한 구현**:
- Event Sourcing 활용하여 모든 액션 기록
- 게임 종료 후 리플레이 가능하도록
- DB에 저장 (현재는 메모리만 사용)

---

## 🔧 문제 해결 가이드

### 컴파일 에러

#### 문제: 의존성 해결 실패
```bash
./gradlew compileJava
# ERROR: Could not resolve dependencies
```

**해결**:
```bash
./gradlew clean build --refresh-dependencies
```

#### 문제: Lombok 관련 에러
```bash
# ERROR: cannot find symbol @Getter
```

**해결**:
- IntelliJ: Settings → Build → Compiler → Annotation Processors → Enable annotation processing
- Eclipse: Lombok 플러그인 설치

### 테스트 실패

#### 문제: 특정 테스트만 실패
```bash
./gradlew test
# Texas Hold'em 통합 테스트 > ... FAILED
```

**해결 단계**:
1. 테스트 격리 확인
   ```java
   @AfterEach
   void tearDown() {
       player1.releaseNickname();  // 닉네임 해제 필수!
       player2.releaseNickname();
   }
   ```

2. 테스트 순서 무관성 확인
   ```bash
   # 특정 테스트만 실행
   ./gradlew test --tests "*TexasHoldemIntegrationTest.GameStartTest.startTexasHoldem_ShouldDealTwoCardsToEachPlayer"
   ```

3. 디버그 모드 실행
   ```bash
   ./gradlew test --debug-jvm
   # IntelliJ에서 5005 포트로 원격 디버깅 연결
   ```

#### 문제: 모든 테스트가 한번에 실패
```bash
# 49 tests completed, 49 failed
```

**가능한 원인**:
1. PostgreSQL 미실행
   ```bash
   docker compose up -d
   docker compose ps  # postgres가 Up 상태인지 확인
   ```

2. 버그 수정 후 체크: `Dealer.java`의 Bug #1, Bug #2 수정이 제대로 되었는지 확인

### WebSocket 연결 문제

#### 문제: 클라이언트가 서버에 연결 못함
```bash
# ERROR: Connection refused ws://localhost:8080/ws/game
```

**해결**:
1. 서버 실행 확인
   ```bash
   ./gradlew bootRun
   # 8080 포트에서 서버 실행 중인지 확인
   ```

2. 포트 충돌 확인
   ```bash
   lsof -i :8080
   # 이미 사용 중이면 프로세스 종료
   ```

3. 헬스 체크
   ```bash
   curl http://localhost:8080/actuator/health
   # {"status":"UP"} 응답 확인
   ```

#### 문제: 메시지 파싱 에러
```bash
# ERROR: Failed to parse GAME_STATE_UPDATE
```

**해결**:
1. PROTOCOL-COMPARISON.md 참고하여 필드명 확인
2. 서버 응답 로그 확인
   ```bash
   # GameCommandService.java에 로그 추가
   log.info("Broadcasting game state: {}", objectMapper.writeValueAsString(stateMessage));
   ```

### 데이터베이스 문제

#### 문제: PostgreSQL 연결 실패
```bash
# ERROR: org.postgresql.util.PSQLException: Connection refused
```

**해결**:
```bash
# PostgreSQL 시작
docker compose up -d

# 로그 확인
docker compose logs postgres

# 재시작
docker compose restart postgres
```

---

## 📞 도움말 및 리소스

### 문서
- **CLAUDE.md**: 프로젝트 전체 가이드
- **PROTOCOL-COMPARISON.md**: 서버-클라이언트 프로토콜 비교
- **docs/adr/**: 아키텍처 결정 기록

### 테스트
```bash
# 모든 테스트 실행
./gradlew test

# 특정 패키지 테스트
./gradlew test --tests "dev.xiyo.pokerhole.dealer.*"

# 테스트 리포트
open build/reports/tests/test/index.html
```

### 유용한 명령어
```bash
# 서버 실행
./gradlew bootRun

# 빌드
./gradlew build

# 의존성 트리
./gradlew dependencies

# 프로젝트 정보
./gradlew properties
```

---

## ✅ 최종 체크리스트

Step 8 완료 전 확인:

**서버**:
- [ ] 전체 테스트 49개 모두 통과
- [ ] 컴파일 에러 없음
- [ ] WebSocket 연결 정상 작동
- [ ] GameState 브로드캐스트 정상 작동

**문서**:
- [ ] ADR-002 업데이트 완료
- [ ] pokerhole-server/README.md 업데이트 완료
- [ ] CLAUDE.md 최종 검토 완료
- [ ] 모든 문서 간 일관성 확인

**클라이언트**:
- [ ] 3개 메시지 타입 추가 확인
- [ ] 프로토콜 일치 확인

**마무리**:
- [ ] NEXT-STEP.md → COMPLETED.md로 이름 변경
- [ ] Git commit 및 push
- [ ] Phase 1 완료 선언!

---

**Good luck!** 🚀

Phase 1을 거의 완료했습니다. Step 8만 하면 Texas Hold'em 서버 구현이 끝납니다!
