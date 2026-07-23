# MySQL 8.0 → 8.4 업그레이드 테스트케이스

작성일: 2026-07-23  
연관 문서: `2026.07.14_RDS_MySQL_8.0_to_8.4_무중단_업그레이드.md`

---

## 테스트 단계 개요

| 단계                            | 환경                               | 상태                |
| ----------------------------- | -------------------------------- | ----------------- |
| PHASE 1. 로컬 연결 테스트            | 로컬 → `vraccountdb-84` (테스트 인스턴스) | ✅ 완료 (2026-07-22) |
| PHASE 2. Dev ECS 배포 검증        | Dev ECS → `vraccountdb-84`       | ✅ 완료              |
| PHASE 3. 운영 In-place 업그레이드 검증 | 운영 → `vraccountdb` (8.4 전환 후)    | ⬜ 진행 필요 (7/28)    |

---

## PHASE 1. 로컬 연결 테스트 ✅ 완료

> 로컬 앱을 `vraccountdb-84` 테스트 인스턴스에 연결해 기본 호환성 확인

### TC-L-001. HikariPool 연결

| 항목 | 내용 |
|------|------|
| 목적 | MySQL 8.4에서 JDBC 연결 풀 정상 초기화 확인 |
| 확인 방법 | 앱 기동 로그 |
| 기대 결과 | `HikariPool-1 - Start completed` 로그 출력 |
| 결과 | ✅ 정상 (`Database version: 8.4.8` 확인) |

### TC-L-002. Flyway 마이그레이션

| 항목 | 내용 |
|------|------|
| 목적 | 기존 37개 마이그레이션 스크립트가 8.4에서 정상 적용되는지 확인 |
| 확인 방법 | 앱 기동 로그 |
| 기대 결과 | `Successfully applied 37 migrations` (또는 `Schema ... is up to date`) |
| 결과 | ✅ 정상 |

### TC-L-003. JPA / Hibernate 초기화

| 항목 | 내용 |
|------|------|
| 목적 | Hibernate Dialect가 MySQL 8.4를 정상 인식하는지 확인 |
| 확인 방법 | 앱 기동 로그에서 ERROR / WARN 검색 |
| 기대 결과 | `MySQLDialect` 관련 오류 없음 |
| 결과 | ✅ 정상 |

### TC-L-004. Spring Batch 메타 테이블

| 항목 | 내용 |
|------|------|
| 목적 | `BATCH_JOB_INSTANCE` 등 Spring Batch 메타 테이블이 8.4에서 정상 동작하는지 확인 |
| 확인 방법 | 앱 기동 후 배치 1회 수동 실행 |
| 기대 결과 | 기동 시 메타 테이블 관련 오류 없음 |
| 결과 | ✅ 정상 |

### TC-L-005. 인증 플러그인 확인

| 항목 | 내용 |
|------|------|
| 목적 | MySQL 8.4에서 비활성화된 `mysql_native_password` 영향 여부 확인 |
| 확인 쿼리 | `SELECT user, plugin FROM mysql.user WHERE user = 'admin';` |
| 기대 결과 | `caching_sha2_password` |
| 결과 | ✅ 변경 완료 (2026-07-22) |

---


## PHASE 2. Dev ECS 배포 검증 ⬜

> Dev ECS가 `vraccountdb-84`에 연결된 상태에서 실제 배치 스케줄이 정상 실행되는지 수일간 확인

### TC-D-001. ECS 태스크 기동

| 항목 | 내용 |
|------|------|
| 목적 | Dev 환경에서 MySQL 8.4 연결 정상 확인 |
| 확인 방법 | ECS → 클러스터 → 태스크 → 로그 탭 |
| 확인 위치 | CloudWatch 로그 |
| 기대 결과 | `HikariPool-1 - Start completed` |
| 결과 | ✅ 정상 (2026-07-23 11:24) |

**확인된 항목 (2026-07-23 11:24 기동 로그 기준):**
- `HikariPool-1 - Start completed` ✅
- `Database version: 8.4.8` ✅
- `Found 41 JPA repository interfaces` ✅ (신규 미션 레포지토리 포함)
- `using meta data indicating: MYSQL` (Spring Batch) ✅
- `중계 서버 활성화: http://relay-dev.test:8090` ✅
- `Started VroongApiApplication in 96.019 seconds` ✅
- 기동 로그 내 ERROR 없음 ✅

> **확인 필요**: 기동 로그 DB URL이 `vraccountdev`로 연결됨. 운영 스냅샷 데이터(`vraccount`)와 동일한 데이터인지 DBeaver에서 확인 권장
> ```sql
> SHOW DATABASES; -- vraccount / vraccountdev 둘 다 있는지 확인
> SELECT COUNT(*) FROM vr_rider_info; -- 운영 데이터 존재 여부 확인
> ```

### TC-D-002. API 정상 응답

| 항목 | 내용 |
|------|------|
| 목적 | 주요 기능이 8.4에서 정상 동작하는지 확인 |
| 확인 대상 | 정산 조회, 라이더 목록, 배달 현황 |
| 확인 방법 | Swagger 또는 Postman |
| 기대 결과 | HTTP 200, 기존과 동일한 응답 |
| 결과 | ⬜ |

### TC-D-003. 배치 그룹별 정상 실행

> 각 배치가 스케줄대로 최소 1회 이상 성공 실행되는지 확인  
> 확인 방법: CloudWatch 로그 또는 아래 DB 쿼리

```sql
-- Spring Batch 실행 결과 확인 (최근 50건)
SELECT JOB_NAME, STATUS, EXIT_CODE, START_TIME, END_TIME
FROM BATCH_JOB_EXECUTION
ORDER BY START_TIME DESC
LIMIT 50;
-- EXIT_CODE = 'COMPLETED' 이어야 함, 'FAILED' 없어야 함
```

| TC          | 배치명                               | 실행 주기           | 우선순위      | 결과                                       |
| ----------- | --------------------------------- | --------------- | --------- | ---------------------------------------- |
| TC-D-003-01 | `hectoDepositProcessJob`          | 매분              | ★ 최우선     | ✅ processedCount: 0 (입금 건 없음, 정상)        |
| TC-D-003-02 | `batchWithdrawalJob`              | 매일 00:05        | ★ 최우선     | ⬜ 운영 전환 당일 스케줄 실행으로 검증                   |
| TC-D-003-03 | `balanceSnapshotJob`              | 매일 00:00        | ★ 최우선     | ⬜ 운영 전환 당일 스케줄 실행으로 검증                   |
| TC-D-003-04 | `riderInfoJob`                    | 매시 :00          | ★ 높음      | ✅ total: 2,282 (2026-07-23)              |
| TC-D-003-05 | `riderInfoIfJob`                  | 매시 :30          | ★ 높음      | ✅ migratedCount: 0 (미이관 데이터 없음, 정상)      |
| TC-D-003-06 | `riderDeliveryHistoriesNowJob`    | 3~10분 간격        | ★ 높음      | ✅ dev 스케줄 실행으로 기확인 (2026-07-23)          |
| TC-D-003-07 | `riderDeliveryHistoriesNowIfJob`  | 3~10분 간격        | ★ 높음      | ✅ dev 스케줄 실행으로 기확인 (2026-07-23)          |
| TC-D-003-08 | `missionCrawlJob`                 | 매 :02/:22/:42   | ★ 높음 (신규) | ✅ dev 스케줄 실행으로 기확인 (2026-07-23)          |
| TC-D-003-09 | `missionRiderProgressCrawlJob`    | 매 :04/:24/:44   | ★ 높음 (신규) | ✅ dev 스케줄 실행으로 기확인 (2026-07-23)          |
| TC-D-003-10 | `baeminCenterMissionIfJob`        | 매 :07/:27/:47   | ★ 높음 (신규) | ✅ dev 스케줄 실행으로 기확인 (2026-07-23)          |
| TC-D-003-11 | `baeminRiderMissionProgressIfJob` | 매 :09/:29/:49   | ★ 높음 (신규) | ✅ dev 스케줄 실행으로 기확인 (2026-07-23)          |
| TC-D-003-12 | `partnerDetailJob`                | 매일 14:00        | 보통        | ✅ 파트너 상세 정보 반환 (2026-07-23)              |
| TC-D-003-13 | `partnerDeliveryFeeJob`           | 매일 09:00, 10:00 | 보통        | ✅ totalCount: 11,661 (2026-07-23)        |
| TC-D-003-14 | `partnerDetailsIfJob`             | 매시 :05          | 보통        | ✅ migratedCount: 83 (2026-07-23)         |
| TC-D-003-15 | `partnerDeliveryFeesIfJob`        | 매시 :05          | 보통        | ✅ migratedCount: 11,661 (2026-07-23)     |
| TC-D-003-16 | `deliveryDailyHistoriesJob`       | 매일 09:00, 10:00 | 보통        | ✅ total: 21, 배달 통계 정상 반환 (2026-07-23)    |
| TC-D-003-17 | `riderDeliveryHistoriesJob`       | 매일 09:00, 10:00 | 보통        | ✅ total: 1,392 (2026-07-23)              |
| TC-D-003-18 | `riderDeliveryHistoriesIfJob`     | 매시 :05          | 보통        | ✅ migratedCount: 0 (미이관 데이터 없음, 정상)      |
| TC-D-003-19 | `deleteOldDataJob`                | 매시 :00          | 낮음        | ✅ 8개 테이블 430,483건 스마트 삭제 완료 (2026-07-23) |

> TC-D-003-02, 03 (`batchWithdrawalJob`, `balanceSnapshotJob`): 금융 처리 배치로 직접 호출 금지. 운영 전환 당일(7/28) 새벽 스케줄 실행 후 CloudWatch 로그로 검증.

### TC-D-004. 이관 배치 transfer_yn 업데이트 검증

> `@Transactional` 수정(2026-07-23) 이후 최초 실제 환경 검증

```sql
-- 미션 이관 정상 여부
SELECT transfer_yn, COUNT(*) AS cnt
FROM baemin_center_mission_if
WHERE if_date = DATE_FORMAT(NOW(), '%Y%m%d')
GROUP BY transfer_yn;
-- transfer_yn = 'Y' 가 존재해야 함

-- 라이더 미션 진척도 이관 정상 여부
SELECT transfer_yn, COUNT(*) AS cnt
FROM baemin_rider_mission_progress_if
WHERE if_date = DATE_FORMAT(NOW(), '%Y%m%d')
GROUP BY transfer_yn;
-- transfer_yn = 'Y' 가 존재해야 함

-- VR 테이블에 실제 데이터 반영 확인
SELECT COUNT(*) FROM vr_baemin_center_mission;
SELECT COUNT(*) FROM vr_baemin_rider_mission_progress;
```

| 항목                                                    | 기대 결과          | 결과  |
| ----------------------------------------------------- | -------------- | --- |
| `baemin_center_mission_if` transfer_yn='Y' 존재         | Y 건수 > 0       | ✅   |
| `baemin_rider_mission_progress_if` transfer_yn='Y' 존재 | Y 건수 > 0       | ✅   |
| `vr_baemin_center_mission` 데이터 증가                     | 전일 대비 동일 또는 증가 | ✅   |
| `vr_baemin_rider_mission_progress` 데이터 증가             | 전일 대비 동일 또는 증가 | ✅   |

### TC-D-005. DB 연결 안정성 (수일간 모니터링)

> 모니터링 시작: 2026-07-23 11:24 / 모니터링 종료 목표: 2026-07-27

| 항목 | 확인 방법 | 기대 결과 | 결과 |
|------|----------|----------|------|
| ConnectionException 없음 | CloudWatch 로그 검색 `ConnectionException` | 미탐지 | ✅ 2026-07-23 기동 이후 미탐지 |
| HikariPool 재연결 오류 없음 | CloudWatch 로그 검색 `HikariPool` + `ERROR` | 미탐지 | ✅ 2026-07-23 기동 이후 미탐지 |
| ECS 태스크 비정상 종료 없음 | ECS → 서비스 → 이벤트 탭 | RUNNING 유지 | ✅ 2026-07-23 기동 이후 RUNNING 유지 |

> 7/28 운영 전환 전까지 매일 CloudWatch 로그 및 ECS 상태 확인 필요.

---

## PHASE 3. 운영 In-place 업그레이드 검증 ⬜ (7/28 예정)

### 전환 전 최종 체크

| 항목 | 확인 내용 | 결과 |
|------|----------|------|
| Dev 검증 완료 | PHASE 2 모든 항목 ⬜ → ✅ | ⬜ |
| 수동 스냅샷 생성 | `vraccountdb-snapshot-before-84-upgrade` Available 상태 | ⬜ |
| 전환 시각 확인 | 04:10 전후 배치 없는 구간 진입 확인 | ⬜ |

### 전환 타이밍 근거

```
04:00 ~ 06:00  riderInfoJob 미실행 (04~06시 제외 조건)
03:31 ~ 05:59  riderDeliveryHistoriesNowJob 미실행
04:09          baeminRiderMissionProgressIfJob 마지막 실행
04:10 ~ 04:20  ★ 가장 한산한 구간 → 업그레이드 시작 권장
04:22          missionCrawlJob 재개 (누락돼도 다음 회차 자동 실행)
```

> `hectoDepositProcessJob` (매분)은 새벽 4시대 실제 입금 건 없어 no-op에 가까움

### TC-P-001. 업그레이드 완료 확인

```sql
-- 버전 확인
SELECT VERSION();
-- 기대: 8.4.x
```

| 항목 | 기대 결과 | 결과 |
|------|----------|------|
| RDS 콘솔 엔진 버전 | 8.4.x | ⬜ |
| `SELECT VERSION()` | 8.4.x | ⬜ |
| 인스턴스 상태 | Available | ⬜ |

### TC-P-002. ECS 재연결 확인

| 항목 | 확인 방법 | 기대 결과 | 결과 |
|------|----------|----------|------|
| HikariPool 재연결 | ECS 로그: `HikariPool-1 - Start completed` | 로그 출력 | ⬜ |
| ECS 태스크 상태 | ECS → 태스크 상태 | RUNNING | ⬜ |
| DB 연결 오류 없음 | ECS 로그 `ConnectionException` 미탐지 | 미탐지 | ⬜ |

### TC-P-003. 전환 후 데이터 정합성

```sql
-- 최근 1시간 데이터 정상 적재 확인
SELECT COUNT(*) FROM vr_rider_acct_weekly
WHERE INS_DATE >= DATE_SUB(NOW(), INTERVAL 1 HOUR);

-- 배치 실행 결과 확인
SELECT JOB_NAME, STATUS, EXIT_CODE, START_TIME
FROM BATCH_JOB_EXECUTION
WHERE START_TIME >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY START_TIME DESC;
```

| 항목 | 기대 결과 | 결과 |
|------|----------|------|
| 최근 1시간 라이더 계정 데이터 존재 | COUNT > 0 | ⬜ |
| 전환 후 첫 배치 COMPLETED | EXIT_CODE = 'COMPLETED' | ⬜ |
| FAILED 배치 없음 | 해당 없음 | ⬜ |

### TC-P-004. 주요 API 정상 응답

| 항목 | 확인 방법 | 기대 결과 | 결과 |
|------|----------|----------|------|
| 정산 조회 API | Swagger / Postman | HTTP 200 | ⬜ |
| 라이더 목록 조회 | Swagger / Postman | HTTP 200 | ⬜ |
| 배달 현황 조회 | Swagger / Postman | HTTP 200 | ⬜ |

---

## 롤백 기준

> 아래 중 하나라도 해당하면 즉시 롤백 결정 (전환 후 30분 이내)

| 조건 | 판단 |
|------|------|
| ECS 태스크가 반복 재시작 (3회 이상) | 즉시 롤백 |
| `BATCH_JOB_EXECUTION` FAILED 건 연속 발생 | 즉시 롤백 |
| `HikariPool` 재연결 반복 실패 | 즉시 롤백 |
| 핵심 API (정산·라이더 조회) HTTP 500 지속 | 즉시 롤백 |

**롤백 절차:**
```
RDS → 스냅샷 → vraccountdb-snapshot-before-84-upgrade
→ 스냅샷 복원 → 인스턴스명: vraccountdb-rollback
→ ECS Task Definition DATABASE_URL → vraccountdb-rollback 엔드포인트로 변경
→ ECS 서비스 강제 재배포
예상 소요: 15~30분
```

---

## 전환 후 1주일 모니터링

| 항목 | 확인 주기 | 확인 방법 |
|------|----------|----------|
| 배치 전체 정상 실행 | 매일 | CloudWatch 로그 or `BATCH_JOB_EXECUTION` |
| DB 연결 수 정상 범위 | 매일 | RDS → 모니터링 → DatabaseConnections |
| ECS 태스크 RUNNING 유지 | 매일 | ECS 서비스 이벤트 탭 |
| 에러 로그 없음 | 매일 | CloudWatch 로그 `ERROR` 검색 |
| `vraccountdb-84` 테스트 인스턴스 삭제 | 7/29 ~ 8/4 | RDS → 삭제 |
