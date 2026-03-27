# DDD 설계: Aggregate 구조

## 현재 상태: Aggregate + Value Object + Domain Event + Domain Policy 확정, Domain Service 미설계

## 확정된 Aggregate 구조

```
┌──────────────────────────────┐
│ Member (AR)                   │
│  - Id                         │
│  - Name                       │
│  - NoShowCount(미처리): int    │  ← 노쇼는 회원의 행동 이력 (PT권이 바뀌어도 유지)
│  - NoShowCount(처리됨): int    │
└──────────────────────────────┘

┌────────────────────────────────────┐
│ Membership (AR)                     │  ← PT권. Member와 1:1
│  - Id                               │
│  - MemberId                         │
│  - TotalSessions: SessionCount (VO) │
│  - Remaining: SessionCount (VO)     │
└────────────────────────────────────┘

┌──────────────────────────────────────┐
│ Trainer (AR)                          │  ← 프로필 + 스케줄을 함께 둠 (동시 충돌 없음)
│  - Id                                 │
│  - Name                               │
│  - SessionDuration: SessionDuration (VO) │
│  - WorkingSchedule: WorkingSchedule (VO) │
└──────────────────────────────────────┘

┌───────────────────────────────┐
│ Reservation (AR)               │  ← 독립 Aggregate. 중복 검증은 Domain Service 담당
│  - Id                          │
│  - MemberId                    │
│  - TrainerId                   │
│  - TimeSlot: TimeSlot (VO)     │
│  - Status: ReservationStatus (enum) │
└───────────────────────────────┘

AR = Aggregate Root, VO = Value Object
```

## 확정된 Value Object 설계

### 시스템 전체 규칙: 분 단위 정규화

모든 시간 관련 VO는 **분 단위**를 최소 정밀도로 사용한다. 초/밀리초는 절삭한다.

| 계층 | 적용 |
|---|---|
| VO 생성자 | 초/밀리초를 0으로 절삭 후 저장 |
| DB 컬럼 | `datetime2(0)` / `time(0)` — DB 레벨에서도 정밀도 일치 |
| 이유 | PT 예약은 분 단위가 최소 의미 단위. 일부 VO만 정규화하면 비교 시 불일치 버그 발생 |

### 1. TimeSlot (예약 시간대)

```csharp
public record TimeSlot
{
    public DateTime Start { get; }
    public DateTime End { get; }

    public TimeSlot(DateTime start, DateTime end)
    {
        Start = TruncateToMinute(start);
        End = TruncateToMinute(end);

        if (End <= Start)
            throw new ArgumentException("종료 시간은 시작 시간 이후여야 합니다");
    }

    public bool OverlapsWith(TimeSlot other)
        => Start < other.End && other.Start < End;

    public TimeSpan Duration => End - Start;

    private static DateTime TruncateToMinute(DateTime dt)
        => new(dt.Year, dt.Month, dt.Day, dt.Hour, dt.Minute, 0);
}
```

- **소속**: Reservation, 슬롯 조회
- **핵심 행동**: `OverlapsWith()` — 예약 중복 검증(Domain Service)에서 호출
- **불변식**: 종료 > 시작, 분 단위 정규화

### 2. SessionDuration (트레이너 세션 길이)

```csharp
public record SessionDuration
{
    public int Minutes { get; }

    public SessionDuration(int minutes)
    {
        if (minutes <= 0)
            throw new ArgumentException("세션 시간은 0보다 커야 합니다");
        if (minutes > 180)
            throw new ArgumentException("세션 시간은 180분을 초과할 수 없습니다");

        Minutes = minutes;
    }

    public TimeSlot CreateSlotFrom(DateTime start)
        => new TimeSlot(start, start.AddMinutes(Minutes));
}
```

- **소속**: Trainer
- **핵심 행동**: `CreateSlotFrom()` — 시작 시간으로부터 슬롯 생성
- **불변식**: 0 < 분 ≤ 180

### 3. DaySchedule (하루 근무 시간)

```csharp
public record DaySchedule
{
    public TimeOnly StartTime { get; }
    public TimeOnly EndTime { get; }

    public DaySchedule(TimeOnly start, TimeOnly end)
    {
        StartTime = TruncateToMinute(start);
        EndTime = TruncateToMinute(end);

        if (EndTime <= StartTime)
            throw new ArgumentException("근무 종료는 시작 이후여야 합니다");
    }

    public bool Contains(TimeOnly time) => time >= StartTime && time < EndTime;

    private static TimeOnly TruncateToMinute(TimeOnly t)
        => new(t.Hour, t.Minute);
}
```

- **소속**: WorkingSchedule 내부
- **핵심 행동**: `Contains()` — 특정 시각이 근무 시간 내인지 판단
- **불변식**: 종료 > 시작, 분 단위 정규화

### 4. WorkingSchedule (주간 근무 스케줄)

```csharp
public record WorkingSchedule
{
    private readonly IReadOnlyDictionary<DayOfWeek, DaySchedule> _schedule;

    public WorkingSchedule(IDictionary<DayOfWeek, DaySchedule> schedule)
    {
        _schedule = new Dictionary<DayOfWeek, DaySchedule>(schedule);
    }

    public bool IsWorkingDay(DayOfWeek day) => _schedule.ContainsKey(day);

    public bool IsWithinWorkingHours(DayOfWeek day, TimeOnly start, TimeOnly end)
    {
        if (!_schedule.TryGetValue(day, out var daySchedule))
            return false;
        return daySchedule.Contains(start) && daySchedule.Contains(end);
    }
}
```

- **소속**: Trainer
- **핵심 행동**: `IsWorkingDay()`, `IsWithinWorkingHours()` — 가용 슬롯 조회 시 사용
- **설계 선택**: Dictionary로 요일별 관리. 근무하지 않는 요일은 키 자체가 없음

### 5. SessionCount (PT 회차)

```csharp
public record SessionCount
{
    public int Value { get; }

    public SessionCount(int value)
    {
        if (value < 0)
            throw new ArgumentException("회차는 0 이상이어야 합니다");

        Value = value;
    }

    public SessionCount Deduct()
        => Value == 0
            ? throw new InvalidOperationException("잔여 회차가 없습니다")
            : new SessionCount(Value - 1);

    public SessionCount Restore() => new SessionCount(Value + 1);

    public bool IsExhausted => Value == 0;
}
```

- **소속**: Membership (TotalSessions, Remaining)
- **핵심 행동**: `Deduct()`, `Restore()`, `IsExhausted`
- **불변식**: 값 ≥ 0, 0일 때 차감 시도 시 예외
- **VO 선택 근거**: `int`로 두면 "0 이하 방지" 검증이 Membership 전체에 흩어짐. 학습 목적으로 VO 경험

### 6. ReservationStatus (예약 상태) — enum

```csharp
public enum ReservationStatus
{
    Confirmed,   // 예약 확정
    Cancelled,   // 취소됨 (24시간 전)
    NoShow,      // 노쇼 처리됨
    Completed    // 정상 완료
}
```

- **VO가 아닌 enum으로 선택한 근거**: 상태 4개, 확장 가능성 낮음. 상태 전이 규칙은 Reservation Entity가 관리

## 확정된 Domain Event 설계

### Event 활용 전략

- **1단계**: Event를 정의하되, 같은 트랜잭션 안에서 동기적으로 Handler 실행
- **2단계**: Outbox 패턴으로 비동기 전환 (알림 등). Handler 코드는 그대로, 전달 방식만 변경
- Event를 지금 정의해두면 Aggregate 간 직접 참조 없이 느슨한 결합을 유지할 수 있음

### Event 목록

#### 1. ReservationCreated (예약 생성됨)

```csharp
public record ReservationCreated(
    Guid ReservationId,
    Guid MemberId,
    Guid TrainerId,
    TimeSlot TimeSlot,
    DateTime OccurredAt
);
```

- **발행**: Reservation.Create()
- **반응**: Membership → 회차 1 차감

#### 2. ReservationCancelled (예약 취소됨 — 24시간 전 정상 취소)

```csharp
public record ReservationCancelled(
    Guid ReservationId,
    Guid MemberId,
    DateTime OccurredAt
);
```

- **발행**: Reservation.Cancel() — CancellationPolicy가 24시간 전 확인 후
- **반응**: Membership → 회차 1 환불

#### 3. NoShowMarked (노쇼 처리됨)

```csharp
public record NoShowMarked(
    Guid ReservationId,
    Guid MemberId,
    DateTime OccurredAt
);
```

- **발행**: Reservation.Cancel() — 24시간 이내 취소 시 / 관리자가 노쇼 마킹 시
- **반응**: Member → 미처리 노쇼 카운트 +1

#### 4. NoShowPenaltyApplied (노쇼 페널티 적용됨)

```csharp
public record NoShowPenaltyApplied(
    Guid MemberId,
    int DeductedSessions,     // 차감된 회차 수 (1)
    int ProcessedNoShowCount, // 처리된 노쇼 건수 (2)
    DateTime OccurredAt
);
```

- **발행**: Member.IncrementUnprocessedNoShow() — 미처리 2회 도달 시
- **반응**: Membership → 1회차 추가 차감, Member → 2건 "처리됨" 전환
- **NoShowMarked와 분리한 이유**: "노쇼 마킹"과 "페널티 적용"은 서로 다른 비즈니스 의미. 분리하면 각각 독립적으로 반응 가능

#### 5. ReservationRescheduled (예약 변경됨)

```csharp
public record ReservationRescheduled(
    Guid ReservationId,
    Guid MemberId,
    TimeSlot OldTimeSlot,
    TimeSlot NewTimeSlot,
    DateTime OccurredAt
);
```

- **발행**: Reservation.Reschedule()
- **반응**: 월별 변경 횟수 추적 (추적 방식은 Domain Policy 설계 시 확정)
- **OldTimeSlot 포함 이유**: 변경 전후를 알아야 감사 로그 및 추적 가능. Event는 사건을 이해하는 데 충분한 정보를 담아야 함

#### 6. ReservationCompleted (예약 완료됨)

```csharp
public record ReservationCompleted(
    Guid ReservationId,
    Guid MemberId,
    Guid TrainerId,
    DateTime OccurredAt
);
```

- **발행**: Reservation.Complete()
- **반응**: 1단계에서는 핸들러 없음. 2단계에서 리뷰 요청, 통계 등 확장 포인트

### Event 흐름도: 발행 → 반응

```
[Reservation Aggregate]
  ├─ Create()      → ReservationCreated
  │                    └→ Membership.DeductSession()
  │
  ├─ Cancel()      → ReservationCancelled         (24시간 전)
  │                    └→ Membership.RestoreSession()
  │
  │                → NoShowMarked                  (24시간 이내)
  │                    └→ Member.IncrementUnprocessedNoShow()
  │                         └→ 2회 누적 시: NoShowPenaltyApplied
  │                              ├→ Membership.DeductSession()
  │                              └→ Member.ProcessNoShows()
  │
  ├─ Reschedule()  → ReservationRescheduled
  │                    └→ (월별 변경 횟수 추적 — Policy 설계 시 확정)
  │
  └─ Complete()    → ReservationCompleted
                       └→ (1단계: 핸들러 없음)
```

## 확정된 Domain Policy 설계

### Policy의 역할

Policy는 **"판단"만** 하고, Entity는 **"실행"만** 한다. 각자 한 가지 책임.

| 로직 | 어디에? | 이유 |
|---|---|---|
| "예약 상태를 Cancelled로 바꾼다" | Entity | 자기 상태 변경 |
| "24시간 전이면 취소 허용, 아니면 노쇼" | Policy | 비즈니스 판단 규칙 |
| "같은 트레이너+겹치는 시간 중복 확인" | Domain Service | 여러 Aggregate 조회 필요 |

### 1. CancellationPolicy (취소 정책)

**규칙**: 예약 24시간 전까지 취소 가능 → 회차 환불. 24시간 이내 → 노쇼 처리.

```csharp
public enum CancellationResult
{
    Allowed,        // 정상 취소 — 회차 환불
    TreatedAsNoShow // 노쇼 처리 — 환불 없음
}

public class CancellationPolicy
{
    private static readonly TimeSpan CancellationDeadline = TimeSpan.FromHours(24);

    public CancellationResult Evaluate(TimeSlot reservationSlot, DateTime now)
    {
        var timeUntilStart = reservationSlot.Start - now;

        return timeUntilStart >= CancellationDeadline
            ? CancellationResult.Allowed
            : CancellationResult.TreatedAsNoShow;
    }
}
```

- Reservation 전체를 받지 않고 TimeSlot만 받음 — 판단에 필요한 최소 정보만 요구
- 24시간이 상수로 분리 — 정책 변경 시 이 클래스만 수정

### 2. NoShowPolicy (노쇼 페널티 정책)

**규칙**: 미처리 노쇼 2회 누적 시 → PT 1회차 추가 차감 + 해당 2건 "처리됨" 전환.

```csharp
public record NoShowPenaltyResult(
    bool ShouldApplyPenalty,
    int SessionsToDeduct,    // 차감할 회차 수
    int NoShowsToProcess     // "처리됨"으로 전환할 노쇼 건수
);

public class NoShowPolicy
{
    private const int PenaltyThreshold = 2;
    private const int PenaltyDeduction = 1;

    public NoShowPenaltyResult Evaluate(int unprocessedNoShowCount)
    {
        if (unprocessedNoShowCount >= PenaltyThreshold)
        {
            return new NoShowPenaltyResult(
                ShouldApplyPenalty: true,
                SessionsToDeduct: PenaltyDeduction,
                NoShowsToProcess: PenaltyThreshold
            );
        }

        return new NoShowPenaltyResult(
            ShouldApplyPenalty: false,
            SessionsToDeduct: 0,
            NoShowsToProcess: 0
        );
    }
}
```

- 결과를 `NoShowPenaltyResult` record로 반환 — 차감 회차, 처리 건수 정보 포함
- `PenaltyThreshold`, `PenaltyDeduction` 상수 분리 — 정책 변경 용이

### 3. ChangePolicy (변경 횟수 제한 정책)

**규칙**: 월 3회까지 예약 변경 가능. 초과 시 거부.

```csharp
public class ChangePolicy
{
    private const int MonthlyChangeLimit = 3;

    public bool CanReschedule(int currentMonthChangeCount)
    {
        return currentMonthChangeCount < MonthlyChangeLimit;
    }
}
```

- `currentMonthChangeCount`는 Domain Service가 Reservation 조회로 계산해서 전달
- 별도 상태(MonthlyChangeCount)를 저장하지 않음 — 동기화 문제 방지

### 변경 횟수 추적 방식 (확정)

Reservation 조회로 count — 별도 상태 관리 불필요.

```
Domain Service → Repository에서 이번 달 변경 건수 조회
              → ChangePolicy.CanReschedule(count)
              → false면 거부, true면 Reservation.Reschedule() 실행
```

### Policy 전체 흐름: 예약 취소 시나리오

```
회원이 예약을 취소한다
  1. CancellationPolicy.Evaluate(timeSlot, now)
     ├─ Allowed → Reservation.Cancel()
     │              → ReservationCancelled 발행
     │              → Handler: Membership.RestoreSession()
     │
     └─ TreatedAsNoShow → Reservation.MarkNoShow()
                            → NoShowMarked 발행
                            → Handler: Member.IncrementUnprocessedNoShow()
                                 → NoShowPolicy.Evaluate(unprocessedCount)
                                      ├─ ShouldApplyPenalty: false → 끝
                                      └─ ShouldApplyPenalty: true
                                           → NoShowPenaltyApplied 발행
                                           → Handler: Membership.DeductSession()
                                           → Handler: Member.ProcessNoShows(2건)
```

### Policy를 인스턴스로 둔 이유

static 메서드로도 충분하지만 인스턴스로 두면:
- 테스트에서 다른 정책으로 교체 가능 (DI)
- 정책 값을 설정에서 읽을 때 생성자 주입으로 자연 확장
- Policy가 독립된 도메인 개념임을 구조로 표현

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
- [x] Value Object 설계 (TimeSlot, SessionDuration, SessionCount, WorkingSchedule, DaySchedule)
- [x] Domain Event 설계 (ReservationCreated, ReservationCancelled, NoShowMarked, NoShowPenaltyApplied, ReservationRescheduled, ReservationCompleted)
- [x] Domain Policy 설계 (CancellationPolicy, NoShowPolicy, ChangePolicy)
- [ ] Domain Service 설계 (ReservationService — 중복 검증)
- [ ] 위 설계를 코드 구조로 매핑 (Clean Architecture 계층)

## Aggregate 경계 판단 기준 (학습 정리)

| 질문 | 같은 Aggregate | 별도 Aggregate |
|---|---|---|
| 동시 변경 시 불일치가 **비즈니스 규칙 위반**인가? | O | |
| 동시 변경 가능하지만 **각자 독립적으로 유효**한가? | | O |
| 현실적으로 **동시 충돌이 빈번**한가? | | 빈번하면 반드시 분리 |
| 현실적으로 **동시 충돌이 거의 없는가?** | 합쳐도 무방 | |
