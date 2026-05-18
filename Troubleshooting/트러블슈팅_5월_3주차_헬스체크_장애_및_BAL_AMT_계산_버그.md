# 트러블슈팅 — 5월 3주차 (2026-05-18)

## 목차
1. [헬스체크 장애 (ECS 태스크 교체 루프)](#1-헬스체크-장애-ecs-태스크-교체-루프)
2. [BAL_AMT 잔액 계산 오류 (REPEATABLE READ)](#2-bal_amt-잔액-계산-오류-repeatable-read)

---

## 1. 헬스체크 장애 (ECS 태스크 교체 루프)

### 발생 일시
- 2026-05-XX 12:05 ~ 12:26 (약 21분)

### 증상
- AWS ECS 콘솔에서 태스크가 unhealthy → 교체 → 또 unhealthy 루프 발생
- ALB 헬스체크 "Request timed out" 반복
- 서비스 일시 장애

---

### 오류 발생 이유

#### 흐름 요약

```
12:00:04  POST /api/v1/migration/migrate (rider_delivery_histories_now_if) 요청 수신
          → migrateTable()
              → processTableMigration()          [REQUIRES_NEW 트랜잭션, DB 커넥션 점유 시작]
                  → migrateRiderDeliveryHistoriesNowIf()
                      → 수백 명 라이더 루프
                          → processRiderDeliveryHistoryNowIndividually() x 수백 회
                              [라이더 1명당 REQUIRES_NEW 트랜잭션 + DB write 2회]

12:05     ALB → GET /actuator/health
              → Spring DataSourceHealthIndicator.SELECT 1
                  → HikariCP 커넥션 획득 대기 → 타임아웃
                      → HTTP 503 / timeout → ALB unhealthy 판정

12:05~12:26  ECS가 unhealthy 태스크를 교체하지만 신규 태스크에서도 동일 배치 재실행 → 루프
```

#### 원인 1 — `processTableMigration`의 불필요한 외부 트랜잭션

```java
// BaeminToVroongIfUseCase.java:166
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public BaeminToVroongIfResult.TableMigrationResult processTableMigration(...)
```

이 메서드가 `REQUIRES_NEW` 트랜잭션을 열면서 **DB 커넥션 1개를 8분 동안 점유**했다. 내부 로직은 모두 개별 `REQUIRES_NEW` 트랜잭션으로 처리되므로 외부 트랜잭션은 실질적으로 아무 역할도 하지 않으면서 커넥션만 붙들고 있었다.

#### 원인 2 — 헬스체크가 DB 커넥션에 의존

```yaml
# application.yml
management:
  health:
    db:
      enabled: true  # Spring Boot DataSourceHealthIndicator 활성화
```

`/actuator/health` 호출 시 Spring Boot가 DB에 `SELECT 1`을 날린다. 배치가 커넥션 풀을 점유 중이면 이 쿼리도 커넥션을 얻지 못해 타임아웃 → ALB unhealthy 판정.

#### 원인 3 — `migrateTable`이 사실상 동기 실행

```java
// BaeminToVroongIfUseCase.java:97
public CompletableFuture<BaeminToVroongIfResult> migrateTable(...) {
    BaeminToVroongIfResult.TableMigrationResult tableResult = processTableMigration(command); // 블로킹
    return CompletableFuture.completedFuture(result); // 이미 끝난 결과를 감싸는 것, 비동기 아님
}
```

`CompletableFuture`를 반환하지만 `completedFuture()`는 비동기가 아니다. HTTP 스레드(`http-nio-8080-exec-6`)에서 수백 건 처리가 전부 완료된 후 wrapping만 하는 구조였다.

---

### 수정 내용 및 이유

#### 수정 1 — `application.yml`: DB 헬스체크 비활성화

```yaml
# 수정 전
health:
  db:
    enabled: true

# 수정 후
health:
  db:
    enabled: false
```

**이유**: 헬스체크를 DB 커넥션 상태와 분리한다. 배치나 트래픽 증가로 커넥션 풀이 일시적으로 포화되어도 ALB는 unhealthy 판정을 내리지 않는다. 설정 파일 1줄 변경으로 즉시 적용 가능하고 코드 변경이 없어 리스크가 가장 낮다.

#### 수정 2 — `processTableMigration`: 외부 `@Transactional` 제거

```java
// 수정 전
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public BaeminToVroongIfResult.TableMigrationResult processTableMigration(...)

// 수정 후
public BaeminToVroongIfResult.TableMigrationResult processTableMigration(...)
```

**이유**: 이 메서드는 테이블명 분기(switch)만 담당하고, 실제 데이터 처리는 각 하위 메서드가 자체 `REQUIRES_NEW` 트랜잭션으로 처리한다. 외부 트랜잭션이 없어도 데이터 일관성에는 변화가 없으며, 8분 동안 커넥션을 점유하던 원인이 제거된다.

---

### 수정 후 리스크

| 항목 | 내용 | 대응 |
|---|---|---|
| DB 다운 시 헬스체크 통과 | DB가 완전히 다운되어도 `/actuator/health`는 200을 반환 | DB 장애는 앱 요청 실패 자체로 감지됨. 별도 DB 모니터링(CloudWatch RDS 알람)으로 보완 |
| 트랜잭션 롤백 범위 변경 | 기존엔 외부 트랜잭션 롤백 시 switch 진입 후 실패 전체를 롤백했으나, 실제로는 각 내부 `REQUIRES_NEW`가 이미 커밋되어 있어 의미가 없었음 | 행동 변화 없음. 오히려 실제 트랜잭션 경계가 명확해짐 |
| 배치 중 헬스체크 맹점 | DB 자체 문제를 헬스체크로 감지하지 못함 | 앱 에러 로그 및 RDS CloudWatch 메트릭으로 보완 권장 |

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

| 항목 | 내용 | 대응 |
|---|---|---|
| Non-Repeatable Read | 같은 트랜잭션 내에서 동일 쿼리를 두 번 실행하면 결과가 다를 수 있음 | `migrateVrRiderAcctDaily`는 previousBalance를 단 1회 조회 후 계산하므로 영향 없음 |
| Phantom Read | 범위 쿼리 결과가 달라질 수 있음 | 해당 메서드는 단건 조회 위주이므로 영향 없음 |

---

## 관련 파일

| 파일 | 변경 내용 |
|---|---|
| `api/src/main/resources/application.yml` | `management.health.db.enabled: false` |
| `api/.../BaeminToVroongIfUseCase.java:166` | `processTableMigration` — `@Transactional` 제거 |
| `api/.../BaeminToVroongIfUseCase.java:1362` | `migrateVrRiderAcctDaily` — `isolation = READ_COMMITTED` 추가 |
