# DDD 설계: Aggregate 구조

## 현재 상태: Aggregate 경계 확정, Value Object / Domain Event 미설계

## 확정된 Aggregate 구조

```
┌──────────────────────┐
│ Member (AR)           │
│  - Id                 │
│  - Name               │
│  - NoShowCount(미처리) │  ← 노쇼는 회원의 행동 이력 (PT권이 바뀌어도 유지)
│  - NoShowCount(처리됨) │
└──────────────────────┘

┌──────────────────┐
│ Membership (AR)   │  ← PT권. Member와 1:1
│  - Id             │
│  - MemberId       │
│  - TotalSessions  │
│  - Remaining      │
└──────────────────┘

┌─────────────────────────┐
│ Trainer (AR)             │  ← 프로필 + 스케줄을 함께 둠 (동시 충돌 없음)
│  - Id                    │
│  - Name                  │
│  - SessionDuration       │
│  - WorkingSchedule       │
└─────────────────────────┘

┌─────────────────────┐
│ Reservation (AR)     │  ← 독립 Aggregate. 중복 검증은 Domain Service 담당
│  - Id                │
│  - MemberId          │
│  - TrainerId         │
│  - TimeSlot (VO)     │
│  - Status            │
└─────────────────────┘

AR = Aggregate Root, VO = Value Object
```

## 설계 결정과 근거

### 1. Reservation을 Trainer에서 분리한 이유
- 인기 트레이너에게 2명이 동시 예약 → 현실적으로 자주 발생
- Trainer 안에 예약 목록을 두면 프로필 수정과 예약 생성이 서로 락 충돌
- 중복 예약 검증(같은 트레이너+겹치는 시간 불가)은 Domain Service가 담당

### 2. Membership을 Member에서 분리한 이유
- 이름 변경과 회차 차감은 관련 없는 변경
- PT권 교체(30회권 소진 → 20회권 구매) 시 독립적으로 관리 가능

### 3. NoShow 카운트를 Member에 둔 이유
- 노쇼는 회원의 행동 이력이지 PT권의 속성이 아님
- PT권이 바뀌어도 노쇼 이력은 유지되어야 함
- 이름 변경과 동시 충돌 가능성? → 한 회원이 동시에 노쇼 2건 불가 → 현실적 충돌 없음

### 4. Trainer에 스케줄을 함께 둔 이유
- 프로필 변경, 스케줄 변경 모두 관리자가 가끔 하는 작업
- 동시 충돌이 현실적으로 없음 → 분리할 이유 부족

## Aggregate 간 협력: 예약 생성 흐름

```
회원이 예약을 생성한다
  1. 잔여 회차 확인 (Membership) → 0이면 거부
  2. 시간 충돌 확인 (Domain Service → Reservation 조회)
  3. 예약 생성 (Reservation)
  4. 회차 차감 (Membership)
  → 1단계: 2,3,4를 한 트랜잭션으로 처리 (실용적 타협)
  → 2단계(알림 추가 시): Outbox 패턴으로 이벤트 기반 전환 검토
```

### "한 트랜잭션" 선택 근거
- DDD 원칙(1 트랜잭션 = 1 Aggregate)에서 벗어나지만:
  - 같은 DB, 모놀리스 → 분산 트랜잭션 불필요
  - 한 회원이 동시에 2건 예약 안 함 → 락 충돌 없음
  - Outbox/Saga는 이 규모에서 과도한 복잡성
- 2단계에서 알림(비동기) 도입 시 이벤트 기반으로 자연스럽게 전환

## 다음 할 일
- [ ] Value Object 설계 (TimeSlot, SessionDuration 등)
- [ ] Domain Event 설계 (ReservationCreated, NoShowMarked 등)
- [ ] Domain Policy 설계 (CancellationPolicy, NoShowPolicy, ChangePolicy)
- [ ] Domain Service 설계 (ReservationService — 중복 검증)
- [ ] 위 설계를 코드 구조로 매핑 (Clean Architecture 계층)

## Aggregate 경계 판단 기준 (학습 정리)

| 질문 | 같은 Aggregate | 별도 Aggregate |
|---|---|---|
| 동시 변경 시 불일치가 **비즈니스 규칙 위반**인가? | O | |
| 동시 변경 가능하지만 **각자 독립적으로 유효**한가? | | O |
| 현실적으로 **동시 충돌이 빈번**한가? | | 빈번하면 반드시 분리 |
| 현실적으로 **동시 충돌이 거의 없는가?** | 합쳐도 무방 | |
