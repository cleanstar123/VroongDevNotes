# 트러블슈팅 — 5월 3주차 (2026-05-18)

## 목차
1. [헬스체크 장애 (ECS 태스크 교체 루프)](#1-헬스체크-장애-ecs-태스크-교체-루프)
2. [BAL_AMT 잔액 계산 오류 (REPEATABLE READ)](#2-bal_amt-잔액-계산-오류-repeatable-read)

---

## 1. 헬스체크 장애 (ECS 태스크 교체 루프)

### 발생 일시
- 2026-05-18 12:05 ~ 12:26 (약 21분)

### 증상
- AWS ECS 콘솔에서 태스크가 unhealthy → 교체 → 또 unhealthy 루프 발생
- ALB 헬스체크 "Request timed out" 반복
- 서비스 일시 장애

---

### 장애 타임라인 (BATCH_STEP_EXECUTION 실측)

```
12:00:04  riderDeliveryHistoriesNowIfStep  34초  COMPLETED  ← 정상
12:00:04  riderDeliveryHistoriesNowStep    21초  COMPLETED  ← 정상

12:03:04  riderDeliveryHistoriesNowIfStep  617초 (10분 17초)  FAILED  ← 장애 원인
12:03:04  riderDeliveryHistoriesNowStep    617초 (10분 17초)  FAILED  ← 장애 원인

12:05     ALB 헬스체크 실패 시작 (커넥션 풀 고갈)

12:12     riderDeliveryHistoriesNowStep  STARTED → [NULL]  (ECS 교체 중 API 죽음)
12:14     riderDeliveryHistoriesNowStep  STARTED → [NULL]  (동일)
12:20     양쪽 스텝 모두 2초 만에 FAILED  (API 아직 불안정)
12:21     양쪽 스텝 모두 0초 FAILED
12:24     양쪽 스텝 모두 0초 FAILED

12:27     정상화 (32초 COMPLETED)
```

---

### 오류 발생 이유

#### 아키텍처 구조

Spring Batch(`vroong-3pl-platform-batch`)가 Quartz 스케줄러로 실행되며, Tasklet이 `vroong-backend`의 HTTP 엔드포인트를 호출한다.

```
Quartz 스케줄러 (10분 간격)
  → RiderDeliveryHistoriesNowIfTasklet
      → POST /api/v1/migration/migrate  [timeout=15분]
          → BaeminToVroongIfUseCase.migrateRiderDeliveryHistoriesNowIf()
```

즉 `BATCH_STEP_EXECUTION`의 소요 시간 = HTTP API 응답 대기 시간 = 실제 이관 처리 시간이다.

#### 근본 원인 — 라이더 1건마다 REQUIRES_NEW 트랜잭션

```java
// 수정 전 — migrateRiderDeliveryHistoriesNowIf() 내부 루프
for (RiderDeliveryHistoriesNowIfEntity riderHistory : deduplicated) {
    BaeminToVroongIfUseCase self = applicationContext.getBean(BaeminToVroongIfUseCase.class);
    self.processRiderDeliveryHistoryNowIndividually(riderHistory, now);
    // 라이더 1명 = REQUIRES_NEW 트랜잭션 1개 = 커넥션 획득 → DB write 2회 → 커넥션 반납
}
```

라이더 수백 명 × (커넥션 획득 + DB write 2회 + 커넥션 반납) = 수 분간 커넥션 풀 지속 사용.

#### 추가 원인 — 불필요한 외부 트랜잭션들

| 메서드                             | 문제                                                     |
| ------------------------------- | ------------------------------------------------------ |
| `migrateTable` (L.96)           | `@Transactional(REQUIRES_NEW)` — 직접 DB 작업 없이 커넥션만 점유   |
| `processTableMigration` (L.166) | `@Transactional(REQUIRES_NEW)` — switch 분기만 하면서 커넥션 점유 |

#### 추가 원인 — 헬스체크가 DB 커넥션에 의존

```yaml
management:
  health:
    db:
      enabled: true  # /actuator/health 호출 시 SELECT 1 실행
```

배치가 커넥션 풀을 소진하면 헬스체크 `SELECT 1`도 커넥션을 얻지 못해 타임아웃 → ALB unhealthy → ECS 교체 루프.

#### 동일 패턴의 다른 위험 메서드

| 메서드 | 위험도 | 이유 |
|---|---|---|
| `migrateRiderDeliveryHistoriesNowIf` | **HIGH** | 라이더 1명마다 REQUIRES_NEW, 이번 장애 직접 원인 |
| `migrateRiderDeliveryHistoriesIf` | **HIGH** | 동일 패턴 |
| `migrateVrRiderAcctDaily` | MEDIUM | 외부 트랜잭션이 루프 전체를 감쌈 |
| `migrateRiderFeeDeductDaily` | MEDIUM | 동일 |
| `migratePartnerDetailsIf` 등 3종 | LOW | 이미 100건 단위 배치 처리 적용됨 |

---

### 수정 내용

#### 수정 1 — `application.yml`: DB 헬스체크 비활성화 (즉시 적용)

```yaml
# 수정 전
management.health.db.enabled: true

# 수정 후
management.health.db.enabled: false
```

헬스체크를 DB 커넥션 상태에서 분리. 배치 중 커넥션 풀이 포화되어도 ALB unhealthy 판정 방지.

#### 수정 2 — `processTableMigration`: 외부 `@Transactional` 제거

```java
// 수정 전
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public BaeminToVroongIfResult.TableMigrationResult processTableMigration(...)

// 수정 후
public BaeminToVroongIfResult.TableMigrationResult processTableMigration(...)
```

switch 분기만 하는 메서드가 커넥션을 전체 이관 시간 동안 점유하던 문제 제거.

#### 수정 3 — `migrateTable`: `@Transactional` 제거

```java
// 수정 전
@Transactional(propagation = Propagation.REQUIRES_NEW)
public CompletableFuture<BaeminToVroongIfResult> migrateTable(...)

// 수정 후
public CompletableFuture<BaeminToVroongIfResult> migrateTable(...)
```

직접 DB 작업이 없는 메서드가 커넥션을 점유하던 문제 제거.

#### 수정 4 — `migrateRiderDeliveryHistoriesNowIf`: 100건 단위 배치 처리로 전환 (근본 해결)

```java
// 수정 전 — 라이더 1명마다 REQUIRES_NEW 트랜잭션
for (RiderDeliveryHistoriesNowIfEntity riderHistory : deduplicated) {
    self.processRiderDeliveryHistoryNowIndividually(riderHistory, now);
}

// 수정 후 — 100건 단위 배치 트랜잭션 + 실패 시 개별 폴백
final int BATCH_SIZE = 100;
for (int i = 0; i < deduplicated.size(); i += BATCH_SIZE) {
    List<RiderDeliveryHistoriesNowIfEntity> batch = deduplicated.subList(i, endIndex);
    try {
        self.processRiderDeliveryHistoryNowBatch(batch, now);  // 100건 = 트랜잭션 1개
    } catch (Exception e) {
        // 배치 실패 시 개별 폴백
        for (RiderDeliveryHistoriesNowIfEntity r : batch) {
            self.processRiderDeliveryHistoryNowIndividually(r, now);
        }
    }
}
```

새로 추가된 `processRiderDeliveryHistoryNowBatch` 메서드가 100건을 단일 트랜잭션으로 처리하고 내부에서 `markAsTransferredByIds`까지 완료한다.

#### 수정 5 — `migrateRiderDeliveryHistoriesIf`: 동일 패턴 적용

수정 4와 동일한 구조로 `processRiderDeliveryHistoryBatch` 메서드를 추가하고 루프를 100건 단위 배치로 전환.

---

### 수정 전후 비교

| 항목 | 수정 전 | 수정 후 |
|---|---|---|
| 트랜잭션 수 (300명 기준) | 300개 | 3개 |
| DB 커넥션 점유 시간 | ~10분 | ~수 초 |
| 헬스체크 영향 | DB 커넥션 고갈 시 실패 | DB 상태 무관 |
| 배치 실패 처리 | 건별 예외 무시 | 배치 실패 → 개별 폴백 |

---

### 수정 후 리스크

| 항목 | 내용 | 대응 |
|---|---|---|
| DB 다운 시 헬스체크 통과 | DB가 완전히 다운되어도 `/actuator/health`는 200 반환 | DB 장애는 앱 요청 실패로 감지. CloudWatch RDS 알람으로 보완 |
| 배치 단위 롤백 범위 확대 | 100건 중 1건 실패 시 배치 전체 롤백 후 개별 폴백 재시도 | 폴백 로직이 있으므로 데이터 유실 없음 |
| 스케줄 이상 미해결 | 12:03에 10분 간격이 아닌 3분 후 재실행된 원인 미확인 | QRTZ_CRON_TRIGGERS에서 스케줄 변경 이력 추가 확인 필요 |

---

### 수정 확인 방법

#### 1. 배포 후 BATCH_STEP_EXECUTION 모니터링

배포 후 다음 배치 실행 시점(10분 간격)에 소요 시간 확인:

```sql
SELECT
    STEP_NAME,
    START_TIME,
    END_TIME,
    STATUS,
    TIMESTAMPDIFF(SECOND, START_TIME, END_TIME) AS duration_sec
FROM BATCH_STEP_EXECUTION
WHERE STEP_NAME IN ('riderDeliveryHistoriesNowIfStep', 'riderDeliveryHistoriesIfStep')
  AND START_TIME >= NOW() - INTERVAL 1 HOUR
ORDER BY START_TIME DESC
LIMIT 10;
```

**기대 결과**: `duration_sec`이 60초 이내로 안정화.
**이전 평균**: 26초 / **이전 최대**: 835초

#### 2. 이관 데이터 정합성 확인

배치 실행 후 이관 누락 없이 처리됐는지 확인:

```sql
-- 오늘 이관 미완료 건 수 확인 (0이어야 정상)
SELECT COUNT(*)
FROM rider_delivery_histories_now_if
WHERE transfer_yn = 'N'
  AND if_date = DATE_FORMAT(NOW(), '%Y%m%d');
```

#### 3. 헬스체크 엔드포인트 직접 확인

배포 후 `/actuator/health` 응답에서 DB 항목이 사라졌는지 확인:

```bash
# DB status 항목이 없으면 정상
curl https://{서버주소}/actuator/health | jq .
```

**기대 응답**:
```json
{
  "status": "UP",
  "components": {
    "redis": { "status": "UP" },
    "diskSpace": { "status": "UP" }
    // "db" 항목 없음
  }
}
```

#### 4. 로그에서 배치 처리 단위 확인

배치 실행 로그에서 건별 로그 대신 배치 단위 로그가 찍히는지 확인:

```
# 수정 전 로그 패턴 (건별)
[이관처리-원본데이터] centerId=..., userId=...  (수백 줄 반복)

# 수정 후 로그 패턴 (배치 단위)
배치 이관 완료: 100/300 (100건)
배치 이관 완료: 200/300 (100건)
배치 이관 완료: 300/300 (100건)
배치 이관 완료: 300건 처리
```

---

## 2. BAL_AMT 잔액 계산 오류 (REPEATABLE READ)

### 증상
- 특정 라이더 RP(주정산) 레코드의 BAL_AMT(잔액)가 잘못 계산됨
- 예시: SEQ 138,173 (RP 20260517) → 실제 BAL=92,039, 기대값=417,409
- 직전 정상 잔액(354,670)이 아닌 더 앞선 레코드의 잔액(29,300)을 사용

### 오류 발생 이유

MySQL 기본 격리 수준은 **REPEATABLE READ**다. `migrateVrRiderAcctDaily`가 `REQUIRES_NEW` 트랜잭션으로 시작하면, **트랜잭션 시작 시점의 스냅샷**으로 모든 SELECT가 고정된다.

```
11:10  SYSTEM이 TI(시간제보험) 레코드 INSERT 및 COMMIT
11:12  RP 배치 트랜잭션 시작 (REPEATABLE READ)
         → previousBalance 쿼리 실행
         → 11:10에 커밋된 TI 레코드가 보이지 않음 (스냅샷이 11:12 이전)
         → TI 이전 레코드의 잔액을 previousBalance로 사용
         → RP의 BAL_AMT = 잘못된 previousBalance + RP금액
```

### 수정 내용 및 이유

```java
// BaeminToVroongIfUseCase.java
// 수정 전
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public BaeminToVroongIfResult.TableMigrationResult migrateVrRiderAcctDaily(...)

// 수정 후
@Transactional(propagation = Propagation.REQUIRES_NEW, isolation = Isolation.READ_COMMITTED, rollbackFor = Exception.class)
public BaeminToVroongIfResult.TableMigrationResult migrateVrRiderAcctDaily(...)
```

**이유**: `READ_COMMITTED`로 변경하면 각 SELECT 실행 시점에 가장 최근에 커밋된 데이터를 읽는다. SYSTEM이 TI를 INSERT/COMMIT한 이후에 RP 배치의 `previousBalance` 쿼리가 실행되면 TI 레코드가 정상적으로 보이고, 올바른 이전 잔액을 사용해 BAL_AMT를 계산한다.

### 수정 후 리스크

| 항목                  | 내용                                      | 대응                                                                |
| ------------------- | --------------------------------------- | ----------------------------------------------------------------- |
| Non-Repeatable Read | 같은 트랜잭션 내에서 동일 쿼리를 두 번 실행하면 결과가 다를 수 있음 | `migrateVrRiderAcctDaily`는 previousBalance를 단 1회 조회 후 계산하므로 영향 없음 |
| Phantom Read        | 범위 쿼리 결과가 달라질 수 있음                      | 해당 메서드는 단건 조회 위주이므로 영향 없음                                         |

### 수정 확인 방법

정산 실행 후 해당 라이더의 `vr_account_mst`에서 SEQ 순서대로 BAL_AMT가 연속적으로 이어지는지 확인:

```sql
SELECT
    SEQ_NO,
    ACCT_TYPE,
    ACCT_AMT,
    BAL_AMT,
    REG_DTM
FROM vr_account_mst
WHERE RIDER_ID = '{라이더ID}'
  AND SUPP_ID  = '{협력사ID}'
ORDER BY SEQ_NO DESC
LIMIT 20;
```

**기대 결과**: 각 행의 `BAL_AMT = 직전 행의 BAL_AMT + 현재 행의 ACCT_AMT` 관계가 성립.

---

## 관련 파일

| 파일 | 변경 내용 |
|---|---|
| `api/src/main/resources/application.yml` | `management.health.db.enabled: false` |
| `api/.../BaeminToVroongIfUseCase.java` | `migrateTable` — `@Transactional` 제거 |
| `api/.../BaeminToVroongIfUseCase.java` | `processTableMigration` — `@Transactional` 제거 |
| `api/.../BaeminToVroongIfUseCase.java` | `migrateRiderDeliveryHistoriesNowIf` — 100건 단위 배치 처리 전환 |
| `api/.../BaeminToVroongIfUseCase.java` | `processRiderDeliveryHistoryNowBatch` — 신규 추가 |
| `api/.../BaeminToVroongIfUseCase.java` | `migrateRiderDeliveryHistoriesIf` — 100건 단위 배치 처리 전환 |
| `api/.../BaeminToVroongIfUseCase.java` | `processRiderDeliveryHistoryBatch` — 신규 추가 |
| `api/.../BaeminToVroongIfUseCase.java` | `migrateVrRiderAcctDaily` — `isolation = READ_COMMITTED` 추가 |
