# NEXT-STEP.md - 다음 개발자를 위한 가이드

**프로젝트**: PokerHole (Event-sourced Texas Hold'em)
**마지막 업데이트**: 2025-10-04
**현재 상태**: Phase 1 & Phase 2 완료 ✅
**테스트**: 72/72 passing (100%)

---

## 🎯 빠른 시작 (Quick Start)

### 서버 실행
```bash
cd pokerhole-server

# PostgreSQL 시작
docker compose up -d

# 서버 실행 (port 8080)
./gradlew bootRun

# 모든 테스트 실행
./gradlew test

# 테스트 결과: 72 tests, 100% pass
```

### 클라이언트 실행
```bash
cd pokerhole-cli

# 클라이언트 실행
go run cmd/poker-client/main.go

# 모든 테스트 실행
go test ./...
```

---

## ✅ 완료된 기능 (Phase 1 + Phase 2)

### Phase 1: 기본 Texas Hold'em
- ✅ 게임 로직 (PRE_FLOP → FLOP → TURN → RIVER → SHOWDOWN)
- ✅ 플레이어 액션 (FOLD, CHECK, CALL, RAISE, ALL_IN)
- ✅ 승자 결정 (HandEvaluator, 21 golden tests)
- ✅ WebSocket 실시간 통신
- ✅ 서버 권한 모델 (Server Authority)
- ✅ 서버-클라이언트 프로토콜 동기화

### Phase 2: 고급 기능
- ✅ **블라인드 시스템** (Small/Big Blind, 설정 가능)
- ✅ **사이드 팟** (ALL_IN 시나리오 처리, 복잡한 다중 팟)
- ✅ **턴 타임아웃** (30초 자동 FOLD, 스레드 안정성)

### 테스트 커버리지
- **총 테스트**: 72개
  - HandEvaluatorTest: 25개 (Golden vectors)
  - TexasHoldemIntegrationTest: 18개
  - BlindsTest: 7개 (Phase 2)
  - SidePotTest: 9개 (Phase 2)
  - TurnTimeoutTest: 7개 (Phase 2)
  - Architecture: 4개
  - 기타: 2개

---

## 🚀 다음 단계 옵션 (Phase 3+)

### Option 1: 스플릿 팟 (Split Pot) 구현 ⭐ 추천

**난이도**: 중간
**예상 시간**: 1-2일
**우선순위**: 높음 (공정성)

**현재 문제**:
- 타이(동일한 패) 시 첫 번째 플레이어가 전체 팟 획득
- 여러 플레이어가 동일한 패를 가질 경우 불공정

**구현 내용**:
1. `HandResult`에 `compareTo()` 로직 강화
2. `Dealer.determinePotWinner()`에서 타이 감지
3. 동일한 패를 가진 플레이어들에게 팟 균등 분배
4. 나머지 칩 처리 (딜러 버튼 기준 첫 번째 플레이어에게)

**예시**:
```
Player A: Ace-High Flush
Player B: Ace-High Flush
Pot: 1000 chips

Result: A gets 500, B gets 500
```

**테스트 필요**:
- 2명 타이
- 3명 타이
- 타이 + 일반 승자 혼합
- 나머지 칩 처리 (1001 chips → 501 + 500)

---

### Option 2: 딜러 버튼 로테이션

**난이도**: 낮음
**예상 시간**: 반나절
**우선순위**: 중간

**현재 문제**:
- 딜러 버튼이 고정됨
- 핸드마다 블라인드 위치가 동일

**구현 내용**:
1. `GameRoom`에 `endHand()` 메서드 추가
2. 핸드 종료 시 `dealerButtonPosition = (dealerButtonPosition + 1) % players.size()`
3. 다음 핸드 시작 시 새로운 위치로 블라인드 베팅

**테스트 필요**:
- 버튼 이동 검증
- 다중 핸드 시퀀스
- 플레이어 탈락 시 버튼 조정

---

### Option 3: 핸드 히스토리 영속화

**난이도**: 중간-높음
**예상 시간**: 3-4일
**우선순위**: 중간 (분석/감사)

**구현 내용**:
1. 이벤트 스토어에 게임 이벤트 저장
2. 핸드 종료 시 전체 기록 DB 저장
3. API 엔드포인트로 히스토리 조회
4. 리플레이 기능

**스키마 예시**:
```sql
CREATE TABLE hand_history (
    hand_id UUID PRIMARY KEY,
    game_id UUID NOT NULL,
    started_at TIMESTAMP NOT NULL,
    ended_at TIMESTAMP,
    final_pot INTEGER,
    winner_id UUID,
    events JSONB NOT NULL
);
```

---

### Option 4: 토너먼트 모드

**난이도**: 높음
**예상 시간**: 1주
**우선순위**: 낮음 (확장 기능)

**구현 내용**:
1. 블라인드 레벨 증가 (매 10분마다 2배)
2. 플레이어 탈락 처리
3. 최종 순위 결정
4. 상금 분배 로직

---

### Option 5: 통합 테스트 & 배포 준비

**난이도**: 중간
**예상 시간**: 2-3일
**우선순위**: 높음 (프로덕션)

**구현 내용**:
1. 서버-클라이언트 E2E 테스트
2. Docker 이미지 빌드
3. CI/CD 파이프라인 구축
4. 모니터링 설정 (Prometheus, Grafana)
5. 성능 벤치마크

---

## 📖 프로젝트 구조

### 서버 (Java/Spring Boot)
```
pokerhole-server/src/main/java/dev/xiyo/pokerhole/
├── core/domain/              # 순수 도메인 로직 (Spring 의존성 없음)
│   ├── game/                # Game aggregate, 라운드 관리
│   │   ├── vo/SidePot.java # Side pot value object
│   │   └── event/          # 도메인 이벤트
│   ├── player/             # Player aggregate
│   └── card/               # Card, Deck, HandEvaluator
│
├── adapter/in/             # Input adapters
│   └── websocket/
│       ├── GameWebSocketHandler.java
│       └── service/
│           ├── GameCommandService.java
│           └── TurnTimeoutService.java  # Phase 2
│
└── dealer/
    └── Dealer.java         # Texas Hold'em 핵심 로직
```

### 핵심 파일

**Dealer.java** (pokerhole-server/src/main/java/dev/xiyo/pokerhole/dealer/Dealer.java)
- 게임 시작: `startTexasHoldem()`
- 액션 처리: `processPlayerAction()`
- 라운드 진행: `progressToNextRound()`
- 승자 결정: `determineWinner()`
- 사이드 팟: `createSidePots()` (Phase 2)

**GameCommandService.java**
- WebSocket 액션 처리
- 타임아웃 관리 (Phase 2)
- 게임 상태 브로드캐스트

**TurnTimeoutService.java** (Phase 2)
- 30초 타임아웃 관리
- ScheduledExecutorService
- 스레드 안정성

---

## 🐛 알려진 제한사항

### 1. 스플릿 팟 미구현
**문제**: 타이 시 팟 분할 안 됨
**해결**: Option 1 참고

### 2. 딜러 버튼 고정
**문제**: 버튼이 이동하지 않음
**해결**: Option 2 참고

### 3. 핸드 히스토리 미영속화
**문제**: 게임 기록이 메모리에만 존재
**해결**: Option 3 참고

### 4. 재연결 처리 미흡
**문제**: 플레이어 연결 끊김 시 복구 어려움
**해결**: WebSocket 재연결 로직 + 게임 상태 재동기화

---

## 🧪 테스트 가이드

### 전체 테스트 실행
```bash
cd pokerhole-server
./gradlew test

# 예상 출력:
# BUILD SUCCESSFUL in 17s
# 72 tests completed, 0 failed
```

### 특정 테스트만 실행
```bash
# 블라인드 테스트
./gradlew test --tests "BlindsTest"

# 사이드 팟 테스트
./gradlew test --tests "SidePotTest"

# 타임아웃 테스트
./gradlew test --tests "TurnTimeoutTest"

# Texas Hold'em 통합 테스트
./gradlew test --tests "TexasHoldemIntegrationTest"
```

### 테스트 리포트 확인
```bash
# HTML 리포트 생성
./gradlew test

# 리포트 열기
open build/reports/tests/test/index.html
```

---

## 🔧 개발 팁

### 1. 새로운 기능 추가 시

**순서**:
1. 도메인 모델 설계 (`core/domain/`)
2. 단위 테스트 작성
3. 도메인 로직 구현
4. 통합 테스트 작성
5. WebSocket 프로토콜 확장 (필요 시)
6. 문서 업데이트

**주의사항**:
- 도메인 레이어는 Spring 의존성 없음 (ArchUnit 검증)
- 모든 상태 변경은 이벤트로 기록
- 서버 권한 모델 준수 (클라이언트는 제안만)

### 2. 버그 발견 시

**Phase 1에서 발견된 중요 버그**:
- **Bug #1**: 베팅 라운드 완료 판단 - 모든 플레이어 액션 확인 누락
- **Bug #2**: CHECK 액션 미기록 - `currentRoundBets`에 기록 안 됨

**교훈**:
- 액션 여부와 베팅 금액은 별개로 관리
- 모든 액션은 명시적으로 기록
- 통합 테스트로 엣지 케이스 검증

### 3. 프로토콜 변경 시

**필수 확인사항**:
1. `/PROTOCOL-COMPARISON.md` 참고
2. 서버와 클라이언트 동시 업데이트
3. 통합 테스트 실행
4. 문서 업데이트

---

## 📚 참고 문서

### 필수 읽기
1. **CLAUDE.md** - 프로젝트 전체 가이드, 아키텍처 설명
2. **COMPLETED.md** - Phase 1 & 2 완료 보고서
3. **PROTOCOL-COMPARISON.md** - 서버-클라이언트 프로토콜 비교
4. **docs/adr/** - 아키텍처 결정 기록 (ADR)
   - ADR-001: Event Sourcing
   - ADR-002: Server Authority (Phase 1 & 2 구현 포함)
   - ADR-005: Deterministic RNG

### 서버
- **pokerhole-server/README.md** - 서버 상세 가이드
- **Texas Hold'em 규칙**: https://www.pokernews.com/poker-rules/texas-holdem.htm

### 클라이언트
- **pokerhole-cli/README.md** - 클라이언트 가이드

---

## 🛠️ 문제 해결 (Troubleshooting)

### 서버가 시작되지 않음
```bash
# PostgreSQL 확인
docker compose ps
docker compose logs postgres

# 포트 충돌 확인
lsof -i :8080
kill -9 <PID>

# PostgreSQL 재시작
docker compose down
docker compose up -d
```

### 테스트 실패
```bash
# Clean build
./gradlew clean build --refresh-dependencies

# 특정 테스트 디버그
./gradlew test --tests "ClassName.testMethod" --debug
```

### WebSocket 연결 실패
```bash
# 서버 헬스 체크
curl http://localhost:8080/actuator/health

# WebSocket 테스트
wscat -c ws://localhost:8080/ws/game
```

---

## 🎯 다음 개발자를 위한 체크리스트

시작하기 전에:
- [ ] Java 23+ 설치 확인
- [ ] Docker 실행 확인
- [ ] PostgreSQL 컨테이너 실행 중
- [ ] 모든 테스트 통과 (`./gradlew test` → 72/72)
- [ ] CLAUDE.md 정독
- [ ] ADR 문서 읽기 (특히 ADR-001, ADR-002)
- [ ] Texas Hold'em 규칙 숙지

새 기능 구현 전에:
- [ ] 기존 코드 패턴 파악 (Hexagonal Architecture)
- [ ] 테스트 먼저 작성 (TDD)
- [ ] 도메인 모델 순수성 유지 (No Spring in domain)
- [ ] 이벤트 소싱 패턴 준수
- [ ] 프로토콜 변경 시 클라이언트 동기화

완료 후:
- [ ] 모든 테스트 통과 (회귀 방지)
- [ ] 문서 업데이트
- [ ] 코드 리뷰 (선택)
- [ ] Git commit

---

## 🚀 추천 다음 단계

**Phase 3 시작을 위한 추천 순서**:

1. **스플릿 팟 구현** (1-2일)
   - 가장 중요한 공정성 이슈
   - 테스트하기 쉬움
   - 기존 코드에 영향 적음

2. **딜러 버튼 로테이션** (반나절)
   - 빠르게 완료 가능
   - 게임 흐름 개선

3. **통합 테스트 & 배포 준비** (2-3일)
   - 프로덕션 준비
   - 성능 검증
   - CI/CD 파이프라인

4. **핸드 히스토리** (3-4일, 선택)
   - 분석/감사 기능
   - 이벤트 소싱 활용

---

**Good luck!** 🎰

질문이 있으면 CLAUDE.md와 COMPLETED.md를 참고하세요.
모든 테스트가 통과하는 것을 확인하고 시작하세요! (72/72 passing)
