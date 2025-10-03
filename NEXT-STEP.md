# NEXT-STEP.md

**프로젝트**: PokerHole (Event-sourced Texas Hold'em)
**마지막 업데이트**: 2025-10-03
**현재 상태**: Phase 1 Step 4 완료 (약 60% 진행)

---

## 📊 현재 상태

### ✅ 완료된 작업

**Phase 1 - 서버 WebSocket 통합** (Step 1-4 완료):
- ✅ PlayerSession 상태 추적 (방/매칭)
- ✅ WebSocket 기반 매칭 알림 시스템
- ✅ GameWebSocketHandler 주요 핸들러 구현
- ✅ GameCommandService 생성 (게임 액션, 채팅 브로드캐스트)
- ✅ 31 tests passing (100% success)
- ✅ Hexagonal architecture 준수

### ⚠️ 미완성 작업

**Phase 1 - Step 5 (선택사항)**:
- ⚠️ WebSocket 통합 테스트 미작성
- ⚠️ GameCommandService 실제 게임 로직 TODO 상태
- ⚠️ GameRoom 브로드캐스트 미구현 (participants private)

---

## 🚨 중요: 서버-클라이언트 불일치 발견

### 현재 서버 게임 구조 (실제)

**GameRoom/Dealer는 단순한 카드 비교 게임**:
- ❌ 턴제 Texas Hold'em이 아님
- ❌ CALL, RAISE, FOLD 등의 액션 처리 로직 없음
- ✅ 카드 5장 나눠주고 → 패 비교 → 승자 결정 (단순)

**문서/ADR과의 불일치**:
- 📄 문서: "Texas Hold'em with CALL/RAISE/FOLD/CHECK"
- 💻 실제: "5-card comparison poker"

### 클라이언트 기대사항 (pokerhole-cli)

**CLAUDE.md 및 ADR-002에 명시**:
- Go 클라이언트는 Texas Hold'em 프로토콜 가정
- WebSocket을 통한 게임 액션 (CALL, RAISE, FOLD, CHECK, ALL_IN)
- 서버 권한 모델 (Server Authority)

---

## 🎯 다음 단계 (필수!)

### ⚠️ CRITICAL: 서버-클라이언트 조율

**다음 작업자는 반드시 다음을 확인하세요**:

1. **게임 로직 일치성 확인** (최우선!)
   ```bash
   # 서버 게임 로직 확인
   cd pokerhole-server
   cat src/main/java/dev/xiyo/pokerhole/dealer/Dealer.java
   cat src/main/java/dev/xiyo/pokerhole/server/room/GameRoom.java

   # 클라이언트 기대사항 확인
   cd ../pokerhole-cli
   cat README.md
   cat internal/domain/game/*.go
   ```

2. **프로토콜 정합성 검증**
   - 서버 WebSocket 메시지 타입: `ServerMessageType`
   - 클라이언트 기대 메시지: pokerhole-cli 문서 참고
   - **매칭되지 않으면 클라이언트 수정 필요!**

3. **결정 사항**:

   **Option A: 서버를 단순 게임으로 유지**
   - GameCommandService의 TODO 제거
   - 클라이언트를 카드 비교 게임용으로 단순화
   - ADR-002 (Server Authority) 업데이트
   - 문서에서 "Texas Hold'em" → "5-Card Poker" 수정

   **Option B: 서버를 Texas Hold'em으로 확장**
   - Dealer에 턴제 로직 추가 (1-2주 소요)
   - GameCommandService TODO 구현
   - GameRoom.participants를 public으로 노출하거나 브로드캐스트 메서드 추가
   - 복잡도 높음, 전체 재설계 필요

4. **클라이언트 통합 테스트** (Option A/B 결정 후)
   ```bash
   # 서버 실행
   cd pokerhole-server
   ./gradlew bootRun

   # 클라이언트 실행 (별도 터미널)
   cd pokerhole-cli
   go run cmd/poker-client/main.go

   # 연결/매칭/게임 플로우 테스트
   ```

---

## 📋 작업 우선순위

### Priority 1: 서버-클라이언트 조율 (1-2일)

**Must Do**:
1. [ ] 서버 GameRoom/Dealer 로직 정밀 분석
2. [ ] 클라이언트 domain/game 로직 정밀 분석
3. [ ] 프로토콜 불일치 목록 작성
4. [ ] Option A vs B 결정 (팀 회의 권장)
5. [ ] 결정에 따라 서버 또는 클라이언트 수정

### Priority 2: 통합 테스트 (Option A/B 결정 후)

**서버 측**:
- [ ] WebSocket 통합 테스트 작성 (NEXT-TASK.md Step 5 참고)
- [ ] 매칭 플로우 E2E 테스트
- [ ] 게임 시작/종료 플로우 테스트

**클라이언트 측**:
- [ ] Golden test 벡터 검증 (21 cases)
- [ ] WebSocket 연결/재연결 테스트
- [ ] 오프라인 모드 테스트

### Priority 3: 문서 정리 (통합 테스트 후)

- [ ] README.md Phase 진행률 업데이트
- [ ] ADR-002 현실화 (실제 구현과 일치시키기)
- [ ] CLAUDE.md 게임 규칙 섹션 수정
- [ ] 서버/클라이언트 README 동기화

---

## 🛠️ 기술적 제약사항

### 서버 (pokerhole-server)

**아키텍처 제약**:
- Hexagonal architecture 엄격히 준수 (ArchUnit 테스트)
- `core/application`은 `adapter`에 의존 불가
- GameCommandService는 `adapter/in/websocket/service`에 위치

**현재 구현 한계**:
- `GameRoom.participants`가 private → 브로드캐스트 불가
- `Dealer`에 턴제 로직 없음 → 게임 액션 처리 불가
- WebSocket 세션 관리가 TCP 세션과 분리됨

### 클라이언트 (pokerhole-cli)

**현재 상태**:
- 0 tests (golden tests 미작성)
- 구조만 존재 (internal/domain, internal/adapter)
- WebSocket 클라이언트 코드 있으나 미테스트

**기대 기능** (CLAUDE.md 기준):
- 21 golden test vectors 통과 (Java ↔ Go parity)
- ed25519 서명된 이벤트
- SQLite 로컬 이벤트 저장
- Bubble Tea TUI

---

## 📖 참고 문서

### 필독 (다음 작업 전)

1. **[CLAUDE.md](/CLAUDE.md)** - 프로젝트 전체 가이드
   - Essential Commands
   - Architecture Deep Dive
   - WebSocket Protocol
   - Golden Test Vectors

2. **[docs/adr/002-server-authority.md](/docs/adr/002-server-authority.md)**
   - 서버 권한 모델 설명
   - 클라이언트-서버 통신 플로우
   - **⚠️ 현재 구현과 불일치 가능성 높음**

3. **[pokerhole-server/README.md](/pokerhole-server/README.md)**
   - 서버 아키텍처
   - WebSocket Protocol
   - 테스트 실행 방법

4. **[pokerhole-cli/README.md](/pokerhole-cli/README.md)**
   - 클라이언트 구조
   - Golden test vectors
   - TUI 구조

### 선택 참고

- **[docs/adr/001-event-sourcing-for-gameplay.md](/docs/adr/001-event-sourcing-for-gameplay.md)** - Event Sourcing 배경
- **[docs/adr/005-deterministic-rng-fairness.md](/docs/adr/005-deterministic-rng-fairness.md)** - 덱 셔플 로직

---

## ⚡ 빠른 시작

### 서버 실행
```bash
cd pokerhole-server

# PostgreSQL 시작
docker compose up -d

# 서버 실행
./gradlew bootRun

# 별도 터미널에서 테스트
./gradlew test
```

### 클라이언트 실행
```bash
cd pokerhole-cli

# 클라이언트 실행
go run cmd/poker-client/main.go

# 테스트 (현재 0개)
go test ./...

# Golden tests (TODO)
go test -v ./tests/golden/
```

---

## 🚀 이상적인 다음 Sprint

**Week 1: 조율 및 결정**
- Day 1-2: 서버/클라이언트 로직 분석
- Day 3: Option A vs B 결정 (팀 미팅)
- Day 4-5: 결정에 따른 코드 수정

**Week 2: 통합 및 테스트**
- Day 1-2: 서버 통합 테스트 작성
- Day 3-4: 클라이언트 golden tests 작성
- Day 5: E2E 통합 테스트 (서버 ↔ 클라이언트)

**Week 3: 문서 및 정리**
- Day 1-2: ADR/README 업데이트
- Day 3-4: 코드 리팩토링
- Day 5: Phase 1 최종 리뷰

---

## 🆘 문제 발생 시

### 서버 관련
```bash
# 로그 확인
tail -f logs/application.log

# 데이터베이스 초기화
docker compose down -v
docker compose up -d

# 테스트 재실행
./gradlew clean test
```

### 클라이언트 관련
```bash
# SQLite DB 삭제
rm ~/.pokerhole/poker.db

# 의존성 재설치
go mod tidy
go clean -cache

# 재실행
go run cmd/poker-client/main.go
```

---

## 📞 연락처 및 리소스

- **GitHub Issues**: [pokerhole/issues](https://github.com/bunnyholes/pokerhole/issues)
- **서버 Repo**: [pokerhole-server](https://github.com/bunnyholes/pokerhole-server)
- **클라이언트 Repo**: [pokerhole-cli](https://github.com/bunnyholes/pokerhole-cli)

---

## 🎓 핵심 교훈

**이 프로젝트에서 배운 것**:

1. **문서 ≠ 코드**: "502 tests"는 존재하지 않았고, "Texas Hold'em"은 실제로 카드 비교 게임이었습니다.

2. **서브모듈 복잡도**: origin/main과 origin/solution이 완전히 다른 프로젝트였습니다.

3. **아키텍처 엄격함**: Hexagonal architecture의 layer 규칙을 어기면 테스트가 실패합니다.

4. **서버-클라이언트 조율의 중요성**: 서버와 클라이언트가 다른 게임을 기대하면 통합 불가능합니다.

**다음 작업자에게**:
- ✅ 문서를 신뢰하되, 코드로 검증하세요
- ✅ 서버와 클라이언트를 동시에 확인하세요
- ✅ 프로토콜 불일치는 초기에 발견하세요
- ✅ CI/CD로 문서-코드 동기화를 자동화하세요

---

**Good luck!** 🚀
