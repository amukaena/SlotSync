# Deep Interview Spec: SlotSync — 헬스장 PT 예약/스케줄 관리 시스템

## Metadata
- Interview ID: slotsync-pt-001
- Rounds: 10
- Final Ambiguity Score: 15%
- Type: greenfield
- Generated: 2026-03-26
- Threshold: 20%
- Status: PASSED
- Tech Stack: .NET Core, DDD, Clean Architecture
- Learning Focus: DDD (Aggregate, Value Object, Domain Event, Policy 패턴)

## Clarity Breakdown
| Dimension | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Goal Clarity | 0.9 | 40% | 0.36 |
| Constraint Clarity | 0.9 | 30% | 0.27 |
| Success Criteria | 0.75 | 30% | 0.225 |
| **Total Clarity** | | | **0.855** |
| **Ambiguity** | | | **14.5%** |

## Goal
헬스장 PT 예약/스케줄 관리 시스템. 회원이 직접 예약하고, 관리자가 조정/취소할 수 있는 복합 사용자 구조. DDD 학습이 주 목적이며, 도메인 모델의 풍부함을 우선한다.

## 사용자 역할
- **회원(Member)**: 트레이너/날짜 기준으로 가용 슬롯 조회, 예약 생성/취소/변경
- **관리자(Admin)**: 예약 조회/조정/취소, 트레이너 스케줄 관리

## 1단계 기능 범위
1. **예약 관리**: 생성, 조회(트레이너 기준 + 날짜 기준), 취소, 변경
2. **PT 회차 관리**: PT권(N회) 구매 → 예약 시 1회 차감
3. **취소/변경 정책**: 시간 기반 취소 + 월별 변경 횟수 제한 + 노쇼 페널티

## 2단계 (향후 확장)
- 알림/리마인더 (이메일/SMS) → 비동기 처리, IHostedService
- UI (Blazor/React)

## Constraints

### 예약 규칙
- 트레이너별 가변 세션 시간 (예: 50분 수업 + 10분 정리)
- 같은 트레이너 + 겹치는 시간대에 중복 예약 불가
- 회원은 트레이너 기준 또는 날짜 기준으로 가용 슬롯 조회 가능

### 취소 정책
- 예약 24시간 전까지 취소 가능 → 회차 환불
- 당일 취소(24시간 이내) → 노쇼로 처리

### 변경 정책
- 월 3회까지 예약 변경 가능
- 초과 시 변경 거부

### 노쇼 정책
- 노쇼 발생 시 회차 차감 없음 (예약 시 이미 차감된 것은 그대로)
- 미처리 노쇼 2회 누적 시 → PT 1회차 추가 차감 + 해당 2건은 "처리됨"으로 전환
- 관리 항목: 미처리 노쇼 카운트, 처리된 노쇼 카운트

### PT 회차 관리
- PT권은 N회 단위로 구매 (예: 30회권)
- 예약 생성 시 1회 차감
- 취소(24시간 전) 시 1회 환불
- 잔여 회차 0이면 예약 생성 불가

### 기술 제약
- .NET Core 기반
- API + 테스트만 (UI 없음)
- EF Core + concurrency token (동시성은 도메인 모델 확립 후 적용)

## Non-Goals
- UI/프론트엔드 (1단계)
- 결제/정산 시스템
- 알림/SMS/이메일 (2단계)
- 트레이너 매칭 추천
- 회원가입/인증 (단순 ID 기반으로 처리)

## Acceptance Criteria
- [ ] 회원이 트레이너 기준으로 가용 슬롯을 조회할 수 있다
- [ ] 회원이 날짜 기준으로 가용 슬롯을 조회할 수 있다
- [ ] 회원이 예약을 생성하면 PT권 회차가 1 차감된다
- [ ] 잔여 회차 0이면 예약 생성이 거부된다
- [ ] 같은 트레이너+겹치는 시간에 중복 예약이 불가능하다
- [ ] 24시간 전 취소 시 회차가 환불된다
- [ ] 24시간 이내 취소는 노쇼로 처리된다
- [ ] 미처리 노쇼 2회 누적 시 PT 1회차가 추가 차감된다
- [ ] 월 3회 초과 변경 시 거부된다
- [ ] 관리자가 예약을 조회/조정/취소할 수 있다
- [ ] 모든 비즈니스 규칙이 단위 테스트로 검증된다

## Assumptions Exposed & Resolved
| Assumption | Challenge | Resolution |
|------------|-----------|------------|
| PT 세션은 1시간 고정 | 실제 헬스장은 가변 시간 (Contrarian) | 트레이너별 가변 세션 시간 채택 |
| 전체 기능을 한 번에 구현 | 범위가 넓으면 학습 효과 저하 (Simplifier) | 1단계/2단계 분리, 알림은 2단계 |
| 노쇼 시 회차 차감 | 사용자 제안으로 구체화 | 노쇼 2회 누적 시 1회 차감, 카운트 관리 |

## Technical Context
- .NET Core Web API
- EF Core (DB 미정 — SQLite for dev, SQL Server for prod 권장)
- Clean Architecture (경험 있음)
- DDD 패턴: Aggregate, Value Object, Domain Event, Domain Policy
- 동시성: EF Core concurrency token (도메인 모델 확립 후 적용)
- 백그라운드 잡: IHostedService (2단계)

## Ontology (Key Entities)
| Entity | 유형 | 핵심 속성 | 관계 |
|--------|------|-----------|------|
| Member | Aggregate Root | Id, Name, NoShowCount(미처리/처리) | → Membership |
| Membership (PT권) | Aggregate Root | Id, MemberId, TotalSessions, RemainingSessions | Member가 소유 |
| Trainer | Aggregate Root | Id, Name, SessionDuration, WorkingSchedule | → TimeSlot |
| Reservation | Aggregate Root | Id, MemberId, TrainerId, TimeSlot, Status | Member+Trainer 참조 |
| TimeSlot | Value Object | StartTime, EndTime | Reservation/Trainer에 포함 |
| CancellationPolicy | Domain Policy | 24시간 규칙, 월 3회 변경 제한 | Reservation에 적용 |
| NoShowPolicy | Domain Policy | 2회 누적 시 1회차 차감 | Member+Membership에 적용 |

## Interview Transcript
<details>
<summary>Full Q&A (10 rounds)</summary>

### Round 1 — Goal Clarity
**Q:** 이 시스템의 주요 사용자는 누구인가요?
**A:** 복합 (회원 + 관리자)
**Ambiguity:** 82%

### Round 2 — Goal Clarity
**Q:** 회원이 예약할 때 구체적으로 어떤 흐름을 거치나요?
**A:** 둘 다 지원 (트레이너 기준 + 날짜 기준)
**Ambiguity:** 74%

### Round 3 — Goal Clarity
**Q:** 예약 생성/조회 외에 핵심 기능은?
**A:** 예약 변경/노쇼 + PT 회차 관리 + 알림/리마인더
**Ambiguity:** 63%

### Round 4 — Constraints (Contrarian Mode)
**Q:** PT 예약의 최소 단위를 1시간 고정으로 잡으려고 하셨는데, 정말 그런가요?
**A:** 가변 시간 (트레이너별)
**Ambiguity:** 54%

### Round 5 — Constraints
**Q:** 예약 취소/변경에 비즈니스 규칙이 있나요?
**A:** 복합 정책 (시간 기반 + 월별 변경 횟수 제한 + 노쇼 페널티)
**Ambiguity:** 48%

### Round 6 — Constraints (Simplifier Mode)
**Q:** 1단계로 반드시 있어야 DDD를 의미있게 배울 수 있는 최소 범위는?
**A:** 예약 + 회차관리 + 취소정책 (알림은 2단계)
**Ambiguity:** 41%

### Round 7 — Constraints
**Q:** PT 회차 관리에서 예약 1건당 1회차가 차감되는 구조인가요? 노쇼 시에도?
**A:** 예약 시 차감, 노쇼는 회차 차감 없이 페널티만
**Ambiguity:** 38%

### Round 8 — Success Criteria
**Q:** 이 프로젝트의 '완성'을 어떻게 판단할까요?
**A:** API + 테스트 (UI 없음)
**Ambiguity:** 29%

### Round 9 — Success Criteria
**Q:** 수락 기준 7개 확인
**A:** 노쇼 횟수 관리 추가 요청 — 미처리 노쇼 2회 시 1회차 차감, 처리/미처리 카운트 관리
**Ambiguity:** 18%

### Round 10 — Constraints
**Q:** 월별 예약 변경 횟수 제한?
**A:** 월 3회, 초과 시 거부
**Ambiguity:** 15%

</details>
