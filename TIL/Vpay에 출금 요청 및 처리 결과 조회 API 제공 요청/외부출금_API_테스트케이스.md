# 외부 출금 API 테스트케이스

> 대상: `POST /api/v1/withdrawals`, `GET /api/v1/withdrawals/{requestId}`, 배치 결과조회
> 배포 후 **운영 서버**에서 순서대로 실행 (모든 헥토 호출은 운영 헥토 `https://gw.settlebank.co.kr`로 실제 전송됨)

---

## 테스트 환경 정보

| 항목        | 값                                                   |
| --------- | --------------------------------------------------- |
| X-API-Key | 부릉 측 협의 후 발급값으로 대체                                  |
| 헥토 URL    | `https://gw.settlebank.co.kr` (운영 전용)               |
| 알림 메일 수신자 | cleanstar1212@gmail.com 외 4명 (`application.yml` 참고) |

---

## 사전 조회

```sql
-- 테스트 후 데이터 정리용 requestId 목록 확인
SELECT SEQ_NO, REQUEST_ID, WITHDRAWAL_ID, STATUS, IS_FINAL, VERSION, RETRY_COUNT, INS_DATE
FROM vpay_external_withdrawal
ORDER BY INS_DATE DESC
LIMIT 20;
```

---

## 공통: 테스트 데이터 초기화

```sql
-- 특정 requestId 삭제 (각 TC 재실행 전 필요 시)
-- ⚠️ 운영 DB 직접 조작 — 테스트 완료 후 반드시 정리
DELETE FROM vpay_external_withdrawal
WHERE REQUEST_ID IN (
    'TEST-REQ-001', 'TEST-REQ-002', 'TEST-REQ-003',
    'TEST-REQ-NEW',
    'E2E-REQ-001',
    'BATCH-REQ-001',
    'NOTREDNO-REQ-001',
    'MAXRETRY-REQ-001'
);
```

---

## TC-01: 정상 출금 접수 → HTTP 202 + ACCEPTED 상태 확인

### 목적
신규 requestId로 출금 접수 시 202가 반환되고 DB에 ACCEPTED 상태로 저장되는지 확인

> **⚠️ 운영 주의사항**: 이 TC는 202 응답 직후 비동기로 운영 헥토(`https://gw.settlebank.co.kr`)에 실제 송금 요청이 전송됩니다.
> 아래 요청에서 사용하는 계좌번호 `1234567890`은 존재하지 않는 가상 계좌이므로 헥토에서 거부되어 최종 상태는 **FAILED**가 됩니다.
> 실제 입금은 발생하지 않지만 운영 헥토에 트래픽이 전송됩니다.

### Step 1 — API 호출

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "TEST-REQ-001",
  "withdrawalId": "TEST-WD-001",
  "transferAmount": "10000",
  "currency": "KRW",
  "bankCode": "020",
  "accountNumber": "1234567890",
  "accountMemo": "테스트출금"
}
```

```
$url = "http://localhost:8080/api/v1/withdrawals"
$headers = @{"X-API-Key" = "local-dev-secret-key-2024"}
$body = '{"requestId":"TEST-REQ-001","withdrawalId":"TEST-WD-001","transferAmount":"1000","currency":"KRW","bankCode":"999","accountNumber":"1234567890","accountMemo":"테스트출금"}'
Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $headers -Body $body

-> 이상한 BankCode라서 실패해야함. -> 확인 완료
```

### Step 2 — 응답 확인

```
HTTP 202 Accepted
{
  "requestId": "TEST-REQ-001",
  "withdrawalId": "TEST-WD-001",
  "transferId": "VT...",        ← VT로 시작하는 20자리
  "status": "ACCEPTED",
  "final": false,
  "version": 1,
  "transferAmount": 10000,
  "currency": "KRW",
  "bankCode": "020",
  "maskedAccountNumber": "******7890",
  "accountMemo": "테스트출금",
  "nextPollAfterSeconds": 5,
  "createdAt": "..."
}
```

### Step 3 — DB 확인

```sql
SELECT SEQ_NO, REQUEST_ID, WITHDRAWAL_ID, TRANSFER_ID, STATUS, IS_FINAL, VERSION,
       MCHT_TRD_NO, HECTO_TRD_NO, RETRY_COUNT, INS_DATE
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'TEST-REQ-001';
```

### Step 4 — 로그 확인

```
INFO  [외부출금] 출금 접수 완료: requestId=TEST-REQ-001, withdrawalId=TEST-WD-001, ...
INFO  [외부출금] 헥토 송금 시작: requestId=TEST-REQ-001, mchtTrdNo=EW...
```

### Step 5 — 비동기 처리 완료 후 최종 상태 확인 (10초 후)

```sql
SELECT REQUEST_ID, STATUS, IS_FINAL, VERSION, RESULT_CODE, RESULT_MESSAGE, COMPLETED_AT
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'TEST-REQ-001';
```

```
GET /api/v1/withdrawals/TEST-REQ-001
X-API-Key: {{X-API-Key}}
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 (202) | 202 |
| `status` (즉시) | ACCEPTED |
| `final` (즉시) | false |
| `version` (즉시) | 1 |
| `maskedAccountNumber` | `******7890` (뒤 4자리만 노출) |
| `nextPollAfterSeconds` (즉시) | 5 |
| `MCHT_TRD_NO` | `EW`로 시작하는 22자리 |
| `IS_FINAL` (즉시) | 0 |
| `status` (10초 후) | FAILED (가상 계좌 `1234567890` → 운영 헥토 거부) |
| `IS_FINAL` (10초 후) | 1 |
| `RESULT_CODE` (10초 후) | 0000 이외 (헥토 거부 코드) |

---

## TC-02: 동일 requestId 재요청 → 멱등성 (기존 결과 반환)

### 목적
TC-01과 동일한 requestId로 동일한 내용을 다시 요청하면 새 레코드를 만들지 않고 기존 결과를 그대로 반환하는지 확인

### 사전 조건
TC-01 완료 상태

### Step 1 — TC-01과 동일한 요청 재전송

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "TEST-REQ-001",
  "withdrawalId": "TEST-WD-001",
  "transferAmount": "10000",
  "currency": "KRW",
  "bankCode": "020",
  "accountNumber": "1234567890",
  "accountMemo": "테스트출금"
}
```

```
Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $headers -Body $body -> 확인 완료 멱등성 정상 동작
```

### Step 2 — DB에서 레코드 중복 생성 여부 확인

```sql
SELECT COUNT(*) AS cnt
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'TEST-REQ-001';
-- 기대값: 1 (중복 생성 없음) -> 확인 완료
```

### 로그 확인

```
INFO  [외부출금] 멱등성 요청 - 기존 결과 반환: requestId=TEST-REQ-001, status=...
-> 확인 완료
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 202 |
| DB 레코드 수 | 1 (기존 레코드 그대로) |
| 응답의 `transferId` | TC-01과 동일한 값 |

---

## TC-03: 동일 requestId + 다른 내용 → 409 IDEMPOTENCY_CONFLICT

### 목적
동일한 requestId지만 금액을 다르게 보냈을 때 409 오류가 반환되는지 확인

### 사전 조건
TC-01 완료 상태

### Step 1 — requestId는 동일, transferAmount만 변경하여 요청

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "TEST-REQ-001",
  "withdrawalId": "TEST-WD-001",
  "transferAmount": "99999",
  "currency": "KRW",
  "bankCode": "020",
  "accountNumber": "1234567890",
  "accountMemo": "테스트출금"
}
```

```
$body03 = '{"requestId":"TEST-REQ-001","withdrawalId":"TEST-WD-001","transferAmount":"99999","currency":"KRW","bankCode":"999","accountNumber":"1234567890","accountMemo":"테스트출금"}'; Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $headers -Body $body03 
-> 409 충돌 오류 확인
```
### Step 2 — 응답 확인

```
HTTP 409 Conflict
{
  "error": {
    "code": "IDEMPOTENCY_CONFLICT",
    "message": "동일한 requestId로 내용이 다른 요청이 존재합니다."
  },
  "traceId": "..."
}
-> 확인 완료
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 409 |
| `error.code` | IDEMPOTENCY_CONFLICT |

---

## TC-04: 동일 withdrawalId + 다른 requestId → 409 WITHDRAWAL_ID_CONFLICT

### 목적
동일한 withdrawalId를 다른 requestId로 재시도하면 중복 송금 방지 오류가 반환되는지 확인

### 사전 조건
TC-01 완료 상태 (TEST-WD-001 이미 존재)

### Step 1 — withdrawalId는 동일, requestId만 변경하여 요청

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "TEST-REQ-NEW",
  "withdrawalId": "TEST-WD-001",
  "transferAmount": "10000",
  "currency": "KRW",
  "bankCode": "020",
  "accountNumber": "1234567890",
  "accountMemo": "테스트출금"
}
```

```
$body04 = '{"requestId":"TEST-REQ-NEW","withdrawalId":"TEST-WD-001","transferAmount":"10000","currency":"KRW","bankCode":"999","accountNumber":"1234567890","accountMemo":"테스트출금"}'; try { Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $headers -Body $body04 } catch { $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream()); $reader.ReadToEnd() }
```

### Step 2 — 응답 확인

```
HTTP 409 Conflict
{
  "error": {
    "code": "WITHDRAWAL_ID_CONFLICT",
    "message": "이미 처리된 withdrawalId입니다."
  },
  "traceId": "..."
}

-> 확인 완료
{"traceId":"93c6e102-5825-4e90-acac-6e77f173204e","error":{"code":"WITHDRAWAL_ID_CONFLICT","message":"동일 원출금 신청 번호에 대한 요청이 이미 존재합니다."}}
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 409 |
| `error.code` | WITHDRAWAL_ID_CONFLICT |
| DB에 TEST-REQ-NEW 레코드 | 없음 |

---

## TC-05: X-API-Key 없음 → 401

### Step 1 — X-API-Key 헤더 없이 요청

```
POST /api/v1/withdrawals
Content-Type: application/json

{ "requestId": "TEST-REQ-005", ... }
```

```
try { Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Body $body } catch { $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream()); $reader.ReadToEnd() }
```

### Step 2 — 응답 확인

```
HTTP 401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "X-API-Key 헤더가 누락되었습니다."
  },
  "traceId": "..."
}

-> 확인 완료
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 401 |
| `error.code` | UNAUTHORIZED |

---

## TC-06: X-API-Key 틀림 → 401

### Step 1 — 잘못된 X-API-Key 헤더로 요청

```
POST /api/v1/withdrawals
X-API-Key: wrong-key-value
Content-Type: application/json

{ "requestId": "TEST-REQ-006", ... }

-> 유효하지 않은 API Key입니다. 확인 완료.
```

```
$wrongHeaders = @{"X-API-Key" = "wrong-key-value"}; try { Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $wrongHeaders -Body $body } catch { $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream()); $reader.ReadToEnd() }
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 401 |
| `error.code` | UNAUTHORIZED |

---

## TC-07: 필수 필드 누락 → 400

### Step 1 — bankCode 없이 요청

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "TEST-REQ-007",
  "withdrawalId": "TEST-WD-007",
  "transferAmount": "10000",
  "currency": "KRW",
  "accountNumber": "1234567890"
}

-> 확인완료
{"success":false,"error":"VALIDATION_FAILED","message":"입력 데이터 유효성 검사에 실패했습니다","details":{"bankCode":"bankCode는 필수입니다."},"timestamp":1792629534.317547400,"path":"uri=/api/v1/withdrawals"}
```

```
$body07 = '{"requestId":"TEST-REQ-007","withdrawalId":"TEST-WD-007","transferAmount":"10000","currency":"KRW","accountNumber":"1234567890"}'; try { Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $headers -Body $body07 } catch { $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream()); $reader.ReadToEnd() }
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 400 |
| `error` | `VALIDATION_FAILED` |
| `details` | 누락된 필드명 + 메시지 포함 |

---

## TC-08: currency != KRW → 400

### Step 1 — currency를 USD로 요청

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "TEST-REQ-008",
  "withdrawalId": "TEST-WD-008",
  "transferAmount": "10000",
  "currency": "USD",
  "bankCode": "020",
  "accountNumber": "1234567890"
}

-> 확인완료
{"success":false,"error":"VALIDATION_FAILED","message":"입력 데이터 유효성 검사에 실패했습니다","details":{"currency":"currency는 KRW만 지원합니다."},"timestamp":1792629679.358267400,"path":"uri=/api/v1/withdrawals"}
```

```
$body08 = '{"requestId":"TEST-REQ-008","withdrawalId":"TEST-WD-008","transferAmount":"10000","currency":"USD","bankCode":"020","accountNumber":"1234567890"}'; try { Invoke-RestMethod -Method POST -Uri $url -ContentType "application/json" -Headers $headers -Body $body08 } catch { $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream()); $reader.ReadToEnd() }
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 400 |
| `error` | `VALIDATION_FAILED` |
| `details` | 누락된 필드명 + 메시지 포함 |

---

## TC-09: 결과 조회 — 존재하지 않는 requestId → 404

### Step 1 — 없는 requestId로 조회

```
GET /api/v1/withdrawals/NONEXISTENT-ID
X-API-Key: {{X-API-Key}}

-> 확인 완료
{"traceId":"ee356c82-47ba-4c01-8ddc-0865e835eebd","error":{"code":"NOT_FOUND","message":"요청을 찾을 수 없습니다: NONEXISTENT-ID"}}
```

```
try { Invoke-RestMethod -Method GET -Uri "http://localhost:8080/api/v1/withdrawals/NONEXISTENT-ID" -Headers $headers } catch { $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream()); $reader.ReadToEnd() }
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| HTTP 응답 코드 | 404 |
| `error.code` | NOT_FOUND |

---

## TC-10: 결과 조회 — 상태별 nextPollAfterSeconds 확인

### 목적
각 상태에서 GET 응답의 `nextPollAfterSeconds` 값이 명세와 일치하는지 확인

### Step 1 — DB에서 상태 직접 설정 후 조회

```sql
-- ACCEPTED 상태 설정
UPDATE vpay_external_withdrawal
SET STATUS = 'ACCEPTED', IS_FINAL = 0
WHERE REQUEST_ID = 'TEST-REQ-001';
```

```
GET /api/v1/withdrawals/TEST-REQ-001
X-API-Key: {{X-API-Key}}
```

같은 방식으로 아래 상태도 반복 확인:

```sql
-- PROCESSING
UPDATE vpay_external_withdrawal SET STATUS = 'PROCESSING', IS_FINAL = 0
WHERE REQUEST_ID = 'TEST-REQ-001';

-- SUCCEEDED
UPDATE vpay_external_withdrawal SET STATUS = 'SUCCEEDED', IS_FINAL = 1
WHERE REQUEST_ID = 'TEST-REQ-001';

-- FAILED
UPDATE vpay_external_withdrawal SET STATUS = 'FAILED', IS_FINAL = 1
WHERE REQUEST_ID = 'TEST-REQ-001';

-- PENDING_CONFIRMATION
UPDATE vpay_external_withdrawal SET STATUS = 'PENDING_CONFIRMATION', IS_FINAL = 0
WHERE REQUEST_ID = 'TEST-REQ-001';
```

```
ACCEPTED 확인:
UPDATE vpay_external_withdrawal SET STATUS = 'ACCEPTED', IS_FINAL = 0 WHERE REQUEST_ID = 'TEST-REQ-001';
Invoke-RestMethod -Method GET -Uri "http://localhost:8080/api/v1/withdrawals/TEST-REQ-001" -Headers $headers

PROCESSING 확인:
UPDATE vpay_external_withdrawal SET STATUS = 'PROCESSING', IS_FINAL = 0 WHERE REQUEST_ID = 'TEST-REQ-001';
Invoke-RestMethod -Method GET -Uri "http://localhost:8080/api/v1/withdrawals/TEST-REQ-001" -Headers $headers

SUCCEEDED 확인:
UPDATE vpay_external_withdrawal SET STATUS = 'SUCCEEDED', IS_FINAL = 1 WHERE REQUEST_ID = 'TEST-REQ-001';
Invoke-RestMethod -Method GET -Uri "http://localhost:8080/api/v1/withdrawals/TEST-REQ-001" -Headers $headers
```

### 판정 기준

| status | `nextPollAfterSeconds` | `final` |
|---|---|---|
| ACCEPTED | 5 | false |
| PROCESSING | 10 | false |
| SUCCEEDED | null | true |
| FAILED | null | true |
| PENDING_CONFIRMATION | null | false |

---

## TC-11: E2E — 정상 송금 흐름 (ACCEPTED → SUCCEEDED)

### 목적
운영 헥토 환경에서 실제 입금이 가능한 계좌로 전체 흐름이 정상 동작하는지 확인

> **⚠️ 운영 주의사항**: 이 TC는 운영 헥토로 실제 1원이 송금됩니다. 사전에 수령 가능한 실제 계좌번호와 은행코드를 준비하세요.
> 테스트 완료 후 DB에서 `E2E-REQ-001` 레코드를 삭제하세요.

### Step 1 — 출금 접수

```
POST /api/v1/withdrawals
X-API-Key: {{X-API-Key}}
Content-Type: application/json

{
  "requestId": "E2E-REQ-001",
  "withdrawalId": "E2E-WD-001",
  "transferAmount": "1",
  "currency": "KRW",
  "bankCode": "{{실제 테스트 계좌 은행코드}}",
  "accountNumber": "{{실제 테스트 계좌번호}}",
  "accountMemo": "E2E테스트"
}
```

응답 202 확인 후 `requestId` 메모

### Step 2 — 비동기 처리 대기 (5초 후)

```
GET /api/v1/withdrawals/E2E-REQ-001
X-API-Key: {{X-API-Key}}
```

### Step 3 — 최종 결과 확인 (SUCCEEDED 또는 PENDING_CONFIRMATION)

```sql
SELECT REQUEST_ID, STATUS, IS_FINAL, HECTO_TRD_NO, RESULT_CODE, COMPLETED_AT
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'E2E-REQ-001';
```

### 로그 확인

```
INFO  [외부출금] 헥토 송금 성공: requestId=E2E-REQ-001, trdNo=...
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| 최종 `status` | SUCCEEDED |
| `IS_FINAL` | 1 |
| `HECTO_TRD_NO` | 헥토 응답 trdNo 값 |
| `RESULT_CODE` | 0000 |
| `COMPLETED_AT` | null 아님 |

---

## TC-12: E2E — 배치 결과조회 흐름 (PENDING_CONFIRMATION → SUCCEEDED)

### 목적
VTIM 발생 시 PENDING_CONFIRMATION 상태가 되고, 배치 호출 후 SUCCEEDED로 전이되는지 확인

> **⚠️ 운영 주의사항**: 직접 INSERT 후 테스트 완료 시 `BATCH-REQ-001` 레코드를 삭제하세요.

### Step 1 — PENDING_CONFIRMATION 상태 직접 삽입

TC-11에서 실제 VTIM이 발생한 건의 DB 레코드를 직접 상태 조작하여 재활용하거나, 아래 SQL로 신규 삽입한다.
ACCOUNT_NUMBER는 AES-256-GCM 암호화된 값이어야 하므로 TC-11 실행 후 생성된 실제 레코드에서 복사한다.

```sql
-- TC-11 실행 후 생성된 레코드의 ACCOUNT_NUMBER 암호화값 확인
SELECT ACCOUNT_NUMBER, MCHT_TRD_NO, HECTO_TRD_NO
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'E2E-REQ-001';
-- 위 쿼리 결과의 ACCOUNT_NUMBER와 HECTO_TRD_NO를 아래 INSERT에 사용
```

```sql
INSERT INTO vpay_external_withdrawal (
    REQUEST_ID, WITHDRAWAL_ID, TRANSFER_ID, STATUS, IS_FINAL, VERSION,
    TRANSFER_AMOUNT, BANK_CODE, ACCOUNT_NUMBER, ACCOUNT_MEMO,
    MCHT_TRD_NO, HECTO_TRD_NO, RETRY_COUNT,
    INS_DATE, INS_USER, UPD_DATE, UPD_USER
) VALUES (
    'BATCH-REQ-001', 'BATCH-WD-001', 'VT00000000000000000001',
    'PENDING_CONFIRMATION', 0, 2,
    10000, '020', '{{E2E-REQ-001의 ACCOUNT_NUMBER 값}}', '배치테스트',
    'EW{{YYYYMMDDHHMMSS}}000001', '{{E2E-REQ-001의 HECTO_TRD_NO 값}}', 1,
    NOW(), 'TEST', NOW(), 'TEST'
);
-- MCHT_TRD_NO 예시: EW20261021120000000001 (EW + 현재일시 + 순번)
```

### Step 2 — 배치 엔드포인트 직접 호출

```
POST /api/v1/external/withdrawals/process-pending
X-API-Secret: {{API-Secret}}
```

### Step 3 — 결과 확인

```sql
SELECT REQUEST_ID, STATUS, IS_FINAL, RETRY_COUNT, LAST_CHECKED_AT, COMPLETED_AT
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'BATCH-REQ-001';
```

### 로그 확인

```
INFO  [외부출금배치] 결과조회 처리 완료: ...
INFO  외부 출금 결과조회: requestId=BATCH-REQ-001, mchtTrdNo=..., resultStatus=21
INFO  외부 출금 SUCCEEDED: requestId=BATCH-REQ-001
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| `status` | SUCCEEDED (헥토 resultStatus=21 기준) |
| `IS_FINAL` | 1 |
| `COMPLETED_AT` | null 아님 |
| 배치 응답 `success_count` | 1 |

---

## TC-13: 배치 — PROCESSING 상태 + hectoTrdNo 없음 → 스킵 + retryCount 증가

### 목적
헥토 호출 전 서버가 중단된 PROCESSING 건은 배치가 스킵하고 retryCount만 올리는지 확인

> **⚠️ 운영 주의사항**: 테스트 완료 후 `NOTREDNO-REQ-001` 레코드를 삭제하세요.

### Step 1 — PROCESSING + hectoTrdNo=NULL 레코드 삽입

ACCOUNT_NUMBER는 TC-11 레코드에서 복사하거나, TC-12의 Step 1 쿼리 결과에서 가져온다.

```sql
INSERT INTO vpay_external_withdrawal (
    REQUEST_ID, WITHDRAWAL_ID, TRANSFER_ID, STATUS, IS_FINAL, VERSION,
    TRANSFER_AMOUNT, BANK_CODE, ACCOUNT_NUMBER, MCHT_TRD_NO,
    HECTO_TRD_NO, RETRY_COUNT, INS_DATE, INS_USER, UPD_DATE, UPD_USER
) VALUES (
    'NOTREDNO-REQ-001', 'NOTREDNO-WD-001', 'VT00000000000000000099',
    'PROCESSING', 0, 2,
    5000, '020', '{{E2E-REQ-001의 ACCOUNT_NUMBER 값}}', 'EW{{YYYYMMDDHHMMSS}}000099',
    NULL, 0,
    NOW(), 'TEST', NOW(), 'TEST'
);
```

### Step 2 — 배치 엔드포인트 호출

```
POST /api/v1/external/withdrawals/process-pending
X-API-Secret: {{API-Secret}}
```

### Step 3 — 결과 확인

```sql
SELECT STATUS, HECTO_TRD_NO, RETRY_COUNT, LAST_CHECKED_AT
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'NOTREDNO-REQ-001';
```

### 로그 확인

```
WARN  외부 출금 배치 스킵 (hectoTrdNo 없음): requestId=NOTREDNO-REQ-001, status=PROCESSING
```

### 판정 기준

| 항목 | 기대값 |
|---|---|
| `STATUS` | PROCESSING (변경 없음) |
| `RETRY_COUNT` | 1 (증가) |
| `LAST_CHECKED_AT` | null 아님 |

---

## TC-14: 배치 — MAX_RETRY_COUNT(10회) 초과 → 알림 메일 발송

### 목적
retryCount가 10회에 도달했을 때 상태가 PENDING_CONFIRMATION으로 유지되고 알림 메일이 발송되는지 확인

> **⚠️ 운영 주의사항**: 테스트 완료 후 `MAXRETRY-REQ-001` 레코드를 삭제하세요.

### Step 1 — retryCount=9인 PENDING_CONFIRMATION 레코드 삽입

ACCOUNT_NUMBER는 TC-11 레코드에서 복사. HECTO_TRD_NO는 배치가 운영 헥토 결과조회를 실제 호출하므로, TC-11에서 생성된 실제 헥토 trdNo를 사용한다 (resultStatus=29 → PENDING_CONFIRMATION 유지 시나리오).

```sql
INSERT INTO vpay_external_withdrawal (
    REQUEST_ID, WITHDRAWAL_ID, TRANSFER_ID, STATUS, IS_FINAL, VERSION,
    TRANSFER_AMOUNT, BANK_CODE, ACCOUNT_NUMBER, MCHT_TRD_NO,
    HECTO_TRD_NO, RETRY_COUNT, INS_DATE, INS_USER, UPD_DATE, UPD_USER
) VALUES (
    'MAXRETRY-REQ-001', 'MAXRETRY-WD-001', 'VT00000000000000000088',
    'PENDING_CONFIRMATION', 0, 2,
    50000, '020', '{{E2E-REQ-001의 ACCOUNT_NUMBER 값}}', 'EW{{YYYYMMDDHHMMSS}}000088',
    '{{E2E-REQ-001의 HECTO_TRD_NO 값}}', 9,
    NOW(), 'TEST', NOW(), 'TEST'
);
```

### Step 2 — 배치 엔드포인트 호출 (resultStatus=29로 응답되는 상황 가정)

```
POST /api/v1/external/withdrawals/process-pending
X-API-Secret: {{API-Secret}}
```

### Step 3 — 결과 확인

```sql
SELECT STATUS, IS_FINAL, RETRY_COUNT, LAST_CHECKED_AT
FROM vpay_external_withdrawal
WHERE REQUEST_ID = 'MAXRETRY-REQ-001';
```

### 로그 확인

```
ERROR 외부 출금 최대 재시도 초과 (관리자 수동 확인 필요): requestId=MAXRETRY-REQ-001, retryCount=10
INFO  외부 출금 알림 메일 전송 완료: requestId=MAXRETRY-REQ-001
```

### 알림 메일 수신 확인

수신 대상 메일함에서 아래 내용 확인:
- 제목: `[Vpay 긴급] 외부 출금 수동 확인 필요 - MAXRETRY-REQ-001`
- 본문 테이블에 `requestId`, `withdrawalId`, `mchtTrdNo`, `transferAmount`, 재시도 횟수 포함

### 판정 기준

| 항목 | 기대값 |
|---|---|
| `STATUS` | PENDING_CONFIRMATION (FAILED로 전환하지 않음) |
| `IS_FINAL` | 0 |
| `RETRY_COUNT` | 10 |
| 알림 메일 발송 | 수신 5명 모두 확인 |

---

## TC-15: 결과 조회 — maskedAccountNumber 마스킹 형식 확인

### 목적
GET 응답의 `maskedAccountNumber`가 뒤 4자리만 노출되는지 확인

### Step 1 — TC-01 접수 후 조회

```
GET /api/v1/withdrawals/TEST-REQ-001
X-API-Key: {{X-API-Key}}
```

### 판정 기준

| 계좌번호 | 기대 `maskedAccountNumber` |
|---|---|
| `1234567890` (10자리) | `******7890` |
| `123456789012` (12자리) | `********9012` |
| `1234` (4자리 이하) | `1234` (마스킹 없음) |

---

## 로그 체크리스트

| 상황 | 레벨 | 메시지 패턴 |
|---|---|---|
| 출금 접수 완료 | INFO | `[외부출금] 출금 접수 완료: requestId=..., mchtTrdNo=...` |
| 헥토 송금 성공 | INFO | `헥토 송금 성공: requestId=..., trdNo=...` |
| VTIM 발생 | WARN | `헥토 VTIM 발생: requestId=..., mchtTrdNo=...` |
| 헥토 송금 실패 | WARN | `헥토 송금 실패: requestId=..., outRsltCd=...` |
| 멱등성 재요청 | INFO | `멱등성 요청 - 기존 결과 반환: requestId=..., status=...` |
| withdrawalId 중복 | WARN | `withdrawalId 중복: withdrawalId=..., requestId=...` |
| 배치 SUCCEEDED | INFO | `외부 출금 SUCCEEDED: requestId=...` |
| 배치 FAILED | INFO | `외부 출금 FAILED: requestId=..., resultStatus=...` |
| 배치 스킵 (trdNo 없음) | WARN | `외부 출금 배치 스킵 (hectoTrdNo 없음): requestId=...` |
| 최대 재시도 초과 | ERROR | `외부 출금 최대 재시도 초과 (관리자 수동 확인 필요): ...` |
| 알림 메일 전송 성공 | INFO | `외부 출금 알림 메일 전송 완료: requestId=...` |
| 알림 메일 전송 실패 | ERROR | `외부 출금 알림 메일 전송 실패: requestId=...` |

---

## API 엔드포인트 요약

| 용도 | Method | URL | 인증 |
|---|---|---|---|
| 출금 접수 | POST | `/api/v1/withdrawals` | X-API-Key |
| 결과 조회 | GET | `/api/v1/withdrawals/{requestId}` | X-API-Key |
| 배치 결과조회 (내부) | POST | `/api/v1/external/withdrawals/process-pending` | X-API-Secret |

---

## 관련 테이블

| 테이블 | 역할 |
|---|---|
| `vpay_external_withdrawal` | 외부 출금 요청 관리 (신규) |
