# 헥토 출금 API 분석 보고서
> 부릉(Vpay) 출금 요청 및 처리 결과 조회 API 제공 요청 대응

---

## 1단계: 현재 출금 흐름

### 진입점

| 엔드포인트                               | 클래스:메서드                                         | 용도               |
| ----------------------------------- | ----------------------------------------------- | ---------------- |
| `POST /api/v1/mob/withdrl/request`  | `MobWithdrlController:187` → `requestWithdrl()` | 사용자 출금 요청        |
| `POST /api/v1/vtim/process-pending` | `VtimPendingController:28` → `processPending()` | VTIM 미확정 건 배치 처리 |

인증: 모든 엔드포인트에 `X-API-Secret` 헤더 (`@SecurityRequirement(name = "API-Secret")`, `MobWithdrlController:38`)

---

### 전체 흐름 (`MobWithdrlUseCase.requestWithdrl()`)

```
[1] VTIM Pending 체크 (219~226)
    └─ 이전 건이 VTIM 상태면 신규 출금 차단

[2] 계좌 정보 조회 (233~234)
    └─ PESSIMISTIC_WRITE 락 (MobWithdrlQueryDslRepository:111, :199)
       → 동시 요청 Race Condition 방지

[3] 출금제한 검증 (240~247)
    └─ P(영구제한) / Y(임시제한) 차단

[4] 수수료 조회 (252~260)
    └─ findWithdrawFeeAmt() 조회, 없으면 BigDecimal.ZERO

[5] 잔액 검증 (274~279)
    └─ 잔액 < 요청금액 + 수수료 → 차단

[6] vr_withdrl_list INSERT (319~343)  ← 잔액 차감 시작
    └─ WD_TYPE='W', BALANCE_VAL = 기존잔액 - reqVal

[7] vr_account_mst INSERT (345~369)
    └─ TRAN_AMT = -reqVal, BAL_AMT = 중간잔액

[8] 수수료 INSERT (371~423)
    └─ 별도 vr_withdrl_list (WD_TYPE='F') + vr_account_mst (ACCT_TYPE='F')
    └─ BAL_AMT = 최종잔액 (reqVal + feeAmt 모두 차감)

[9] mchtTrdNo 생성 + PENDING 로그 저장 (439~454)
    └─ 형식: YYYYMM + 5자리 일련번호
    └─ vr_withdraw_request_his에 STATUS='PENDING'
    └─ REQUIRES_NEW 독립 트랜잭션 → 헥토 호출 실패해도 기록 유지
    └─ ** mchtTrdNo는 송금 전에 DB에 저장됨 (확인) **

[10] 헥토 API 호출 (467~470)
     └─ HectoPaymentWebClient.requestWithdrawal()
     └─ URL: {HECTO_RELAY_URL}/relay/hecto/withdrawal (릴레이) 또는
            {HECTO_API_URL}/pyag/v1/fxTrnsmit (직접)
     └─ 요청: mchtId, mchtTrdNo, bankCd, custAcntNo(AES256-ECB 암호화), amt(암호화)

[11-A] 성공 (outRsltCd="0000") → STATUS='COMPLETED'
[11-B] 실패 (outRsltCd≠"0000", ≠"VTIM") → STATUS='FAILED' + 원거래 rollback
[11-C] VTIM (outRsltCd="VTIM")
        └─ vr_withdraw_vtim_pending INSERT (REQUIRES_NEW)
        └─ 원거래 rollback → vr_withdrl_list/vr_account_mst 취소
        └─ 배치 스케줄러(/api/v1/vtim/process-pending)가 이후 처리
```

---

### 상태값 (String 상수, Enum 없음)

**vr_withdraw_request_his.STATUS:**

| 값 | 의미 |
|---|---|
| `PENDING` | 헥토 호출 전 저장 |
| `COMPLETED` | 송금 성공 확정 |
| `FAILED` | 송금 실패 확정 |
| `VTIM` | 타임아웃, vr_withdraw_vtim_pending으로 이관 |

**vr_withdraw_vtim_pending.STATUS:**

| 값 | 의미 |
|---|---|
| `PENDING` | 결과 조회 대기 중 |
| `SUCCESS` | 헥토 status=21, 원장 재반영 완료 |
| `FAILED` | 헥토 status=22/23 |
| `EXPIRED` | 10회 재시도 초과, 관리자 메일 발송 |

VTIM 처리 판단 근거 (`VtimPendingCheckUseCase`):
- `status=21` → SUCCESS + 원장 재반영 (`VtimPendingLedgerService.createLedgerAndMarkSuccess()`)
- `status=22/23` → FAILED
- `status=29` → PENDING 유지, retryCount++
- 10회 초과 → EXPIRED + 메일 (`vroong.notification.vtim.recipients`)

---

### 관련 테이블 요약

| 테이블 | 역할 | 비고 |
|---|---|---|
| `vr_withdrl_list` | 입출금 내역 원장 | WD_TYPE: W(출금), D(입금), F(수수료) |
| `vr_account_mst` | 잔액 원장 | TRAN_AMT 음수로 차감 |
| `vr_withdraw_request_his` | 헥토 거래 기록 | REQUIRES_NEW 독립 커밋 |
| `vr_withdraw_vtim_pending` | VTIM 미확정 관리 | REQUIRES_NEW 독립 커밋 |

계좌번호 저장: AES-GCM 암호화 (`AesEncryptionUtils.encrypt()`), 헥토 전송 시 AES256-ECB 추가 암호화 확인.
로그 마스킹: `MobWithdrlUseCase:458~465` — 계좌번호 `"***"` 처리 확인.
비관적 락: `MobWithdrlQueryDslRepository:111, :199` — `PESSIMISTIC_WRITE` 확인.

---

### 헥토 환경별 URL

| 환경 | API URL | 출처 |
|---|---|---|
| 테스트 | `https://tbgw.settlebank.co.kr` | `application-local.yml` 주석 |
| 운영 | `https://gw.settlebank.co.kr` | `application-local.yml:76` |
| 릴레이 서버 | `http://3.36.153.113:8090` | `application-local.yml:73` |

경로: 출금 `/pyag/v1/fxTrnsmit`, 결과조회 `/pyag/v1/fxResult`

---

## 2단계: Gap 분석

| 요구사항                                                          | 현재 상태     | 근거                                                               | 신규 개발/수정 필요 내용                                               | 난이도   |
| ------------------------------------------------------------- | --------- | ---------------------------------------------------------------- | ------------------------------------------------------------ | ----- |
| `POST /api/v1/withdrawals` 신규 엔드포인트                           | **없음**    | 현재 `/api/v1/mob/withdrl/request`만 존재                             | 신규 Controller + UseCase 작성                                   | 중     |
| `requestId` 저장 및 멱등성 처리                                       | **없음**    | `vr_withdraw_request_his`에 requestId 컬럼 없음                       | 테이블에 `REQUEST_ID VARCHAR UNIQUE` 추가 + 중복 검사 로직               | 중     |
| `withdrawalId` (부릉 원번호) 저장                                    | **없음**    | 현재 테이블에 외부 withdrawalId 컬럼 없음                                    | `WITHDRAWAL_ID` 컬럼 추가 + 유니크 인덱스                              | 하     |
| HTTP 202 즉시 반환 (비동기 처리)                                       | **없음**    | 현재 헥토 응답까지 동기 대기                                                 | 접수 저장 후 202 반환, 헥토 호출은 별도 처리로 분리                             | **상** |
| 잔액·수수료 차감 없이 송금만 수행                                           | **없음**    | 현재 [6]~[8] 잔액차감이 헥토 호출보다 먼저 실행됨                                  | 신규 UseCase에서 [6]~[8] 단계를 건너뛰는 분기. 단, DB 흐름 기록용 INSERT는 유지 필요 | **상** |
| DB에 예치금 흐름 연속 기록 (차감 없이)                                      | **없음**    | 현재 잔액 차감과 흐름 기록이 동일 테이블에 묶여 있음                                   | 부릉 요청용 별도 타입(WD_TYPE='X' 등)으로 INSERT하되 BAL_AMT 미변경 방식 설계 필요  | 중     |
| `GET /api/v1/withdrawals/{requestId}`                         | **없음**    | 현재 결과 조회 API 없음                                                  | 신규 Controller + requestId 기준 조회 Repository                   | 하     |
| 응답: `version`, `final`, `nextPollAfterSeconds`                | **없음**    | 현재 테이블에 없는 필드                                                    | 컬럼 추가 또는 응답 계산 로직                                            | 하     |
| 응답: `maskedAccountNumber`                                     | **부분**    | 저장은 암호화됨. 복호화 후 마스킹 처리 필요                                        | 복호화 후 앞자리 마스킹 변환 로직 추가                                       | 하     |
| 상태값 ACCEPTED/PROCESSING/PENDING_CONFIRMATION/SUCCEEDED/FAILED | **부분**    | 현재 PENDING/COMPLETED/FAILED/VTIM. 이름·의미 불일치                      | 상태 Enum 신규 정의 + 매핑                                           | 하     |
| mchtTrdNo 송금 전 DB 저장                                          | **있음** ✓  | `MobWithdrlUseCase:439~454` REQUIRES_NEW로 저장                     | 불필요                                                          | -     |
| VTIM 발생 시 즉시 실패 처리 금지                                         | **있음** ✓  | `VtimPendingCheckUseCase` — PENDING 유지 후 재조회 최대 10회              | EXPIRED 상태 설명 메일 답장에 포함 필요                                   | -     |
| ⚠️ EXPIRED(10회 초과) 후 자동 실패 처리                                 | **위험 지점** | EXPIRED로 상태 변경, 재송금은 없음. 단 FAILED 확정도 아님 → 부릉이 FAILED로 간주 가능성 있음 | EXPIRED 상태를 부릉 API에서 어떻게 노출할지 정책 결정 필요                       | 중     |
| 동일 requestId + 다른 내용 → 오류 반환                                  | **없음**    | requestId 개념 자체 없음                                               | requestId 저장 후 내용 해시 비교 로직                                   | 중     |
| 동일 withdrawalId + 다른 requestId → 중복 송금 차단                     | **없음**    | withdrawalId 개념 없음                                               | WITHDRAWAL_ID UNIQUE 제약 + 재요청 시 기존 결과 반환                     | 중     |
| DB 유니크 제약 + 선점 처리                                             | **부분**    | PESSIMISTIC_WRITE 락 있음. 단 requestId/withdrawalId 유니크 제약 없음       | REQUEST_ID, WITHDRAWAL_ID에 UNIQUE 인덱스 추가                     | 하     |
| 서버 재시작 후 미처리 이어 처리                                            | **있음** ✓  | `vr_withdraw_vtim_pending` + `/api/v1/vtim/process-pending` 배치   | 신규 ACCEPTED/PROCESSING 건도 재시작 후 처리 로직 필요                     | 중     |
| 계좌번호 암호화 저장                                                   | **있음** ✓  | `MobWithdrlUseCase:291~300` AES-GCM                              | 불필요                                                          | -     |
| 로그 마스킹                                                        | **있음** ✓  | `MobWithdrlUseCase:458~465` `"***"`                              | 불필요                                                          | -     |
| 서버 간 인증 (부릉 전용 별도 수단)                                         | **부분**    | X-API-Secret 있음. 단 부릉 전용 클라이언트 식별 수단 없음                          | 부릉 전용 API Key 발급 또는 별도 헤더 인증 추가                              | 중     |
| 인증된 클라이언트가 자기 거래만 조회                                          | **없음**    | 클라이언트 ID와 거래 연결 개념 없음                                            | 거래 테이블에 `CLIENT_ID` 컬럼 + 조회 시 필터링                            | 중     |
| 오류 코드 표준화 (IDEMPOTENCY_CONFLICT 등)                            | **없음**    | 현재 표준화된 에러 코드 없음                                                 | 표준 에러 응답 DTO + 에러 코드 정의                                      | 중     |
| traceId 포함 오류 응답                                              | **확인 불가** | MDC/traceId 설정 여부 미확인                                            | 확인 후 추가                                                      | -     |
| 권장 재조회 대기 시간 (`nextPollAfterSeconds`)                         | **없음**    | 없음                                                               | 응답 생성 시 상태별 값 계산 로직                                          | 하     |

---

### 핵심 3개 항목 별도 검토

**① 잔액·수수료 차감 없이 "송금만" 분리 가능한가?**

기술적으로 가능하나 영향 범위가 큼. 현재 `MobWithdrlUseCase.requestWithdrl()`에서 [6]~[8](잔액차감) → [9](mchtTrdNo 저장) → [10](헥토 호출)이 한 트랜잭션 안에 묶여 있음. 분리 방식:
- 기존 코드 변경 없이 신규 UseCase를 병렬로 추가 권장
- 부릉 요청용 별도 레코드 타입 정의 (기존 WD_TYPE과 구분)
- 기존 `MobWithdrlUseCase` 로직에는 영향 없음

**② 같은 withdrawalId로 requestId만 바꿔 재요청 시 중복 송금 방지 가능한가?**

현재 불가. `vr_withdraw_request_his`에 withdrawalId 컬럼 자체가 없음. `WITHDRAWAL_ID` 컬럼 + UNIQUE 제약 추가 후, 재요청 시 기존 레코드 조회 → 기존 결과 반환하는 로직이 필요함.

**③ 미확정 거래 자동 실패 처리 위험 지점**

`VtimPendingCheckUseCase` — 10회 재시도 후 `EXPIRED` 처리. 재송금은 발생하지 않음(확인). 단, `EXPIRED`는 FAILED 확정이 아니라 "관리자 확인 대기" 상태. 부릉이 `EXPIRED`를 `FAILED`로 해석하면 미지급 처리가 될 수 있으므로, **상태 노출 방식과 처리 정책을 메일 답장에서 명확히 해야 함**.

---

## 3단계: 답장용 확인사항 초안

> 메일 7번의 8개 항목 기준

---

**1. 개발 가능 여부와 예상 일정**

> 내부 결정 필요.
> 기술적으로는 개발 가능합니다. 다만 기존 출금 흐름(잔액 차감 포함)과 신규 흐름(송금 전용)을 병렬로 운영해야 하므로 신규 API 설계 + 테이블 변경이 필요합니다. 구체적인 일정은 내부 검토 후 회신 드리겠습니다.

---

**2. 잔액·수수료 중복 차감 없이 송금만 수행 가능한지**

> 가능합니다. 전달받는 금액이 최종 입금액이므로, Vpay 서버 내부에서 잔액 차감 및 수수료 계산은 수행하지 않고 헥토 송금만 실행하도록 분리 구현 예정입니다. 단, 예치금 흐름 추적을 위한 DB 기록은 별도 유형으로 유지됩니다.

---

**3. 운영 및 테스트 API 주소**

> Vpay 서버 주소: **내부 결정 필요** (도메인/IP 확인 후 전달)
>
> 연동 대상 헥토 환경:
> - 테스트: `https://tbgw.settlebank.co.kr`
> - 운영: `https://gw.settlebank.co.kr`
>
> 테스트 연동 시 헥토 테스트 계정(mchtId, 암호화 키) 별도 협의 필요.

---

**4. 인증 방식과 인증 정보 전달 방법**

> 현재 Vpay 서버 내부 API에는 `X-API-Secret` 헤더 방식이 적용되어 있습니다. 외부 서버 간 연동을 위해 부릉 전용 API Key 발급 방식으로 추가 인증을 적용할 예정입니다. 정확한 인증 정보는 개발 완료 후 안전한 채널을 통해 전달 드리겠습니다.

---

**5. 확정된 요청·응답 명세**

요청 필드 수용 가능 여부:

| 필드 | 수용 | 비고 |
|---|---|---|
| `requestId` | ✓ | UNIQUE 제약으로 저장, 멱등성 키로 사용 |
| `withdrawalId` | ✓ | UNIQUE 제약, 동일 건 중복 방지 |
| `transferAmount` | ✓ | 수수료 차감 없이 그대로 헥토에 전달 |
| `currency` | ✓ | KRW 고정 수용 |
| `bankCode` | ✓ | 앞자리 0 포함 문자열 처리 |
| `accountNumber` | ✓ | AES-GCM 암호화 저장, 로그 마스킹 |
| `accountMemo` | 확인 필요 | 헥토 `custAcntSumry` 필드 길이 제한 확인 필요 |

응답 필드 제공 가능 여부:

| 필드 | 제공 가능 | 비고 |
|---|---|---|
| `requestId`, `withdrawalId` | ✓ | |
| `transferId` (내부 거래번호) | ✓ | mchtTrdNo 또는 신규 UUID |
| `status` | ✓ | ACCEPTED/PROCESSING/PENDING_CONFIRMATION/SUCCEEDED/FAILED |
| `final` | ✓ | 상태 기반 계산 |
| `version` | ✓ | 업데이트 횟수 카운터 |
| `transferAmount`, `createdAt`, `updatedAt` | ✓ | |
| `completedAt` | ✓ | 성공 확정 시각 |
| `maskedAccountNumber` | ✓ | 복호화 후 마스킹 처리 |
| `nextPollAfterSeconds` | ✓ | 상태별 고정값 제공 예정 |

---

**6. 처리 상태 및 오류 코드 정의**

상태 코드: 제안하신 5개 값(ACCEPTED / PROCESSING / PENDING_CONFIRMATION / SUCCEEDED / FAILED) 수용 예정.

> **PENDING_CONFIRMATION 주의사항**: 헥토 타임아웃(VTIM) 또는 통신 단절 시 이 상태로 유지하며, 내부 배치가 자동으로 결과를 재조회합니다. 재조회 최대 횟수 초과 시 PENDING_CONFIRMATION이 장기간 유지될 수 있으며, 이 경우 FAILED로 자동 확정하지 않고 관리자 확인이 필요한 상태(EXPIRED)로 분리 관리됩니다. **이 상태의 처리 정책(자동 FAILED 전환 여부, 타임아웃 기준)을 협의 부탁드립니다.**

오류 코드 정의안:

| 코드 | HTTP | 설명 |
|---|---|---|
| `VALIDATION_ERROR` | 400 | 입력값 오류 |
| `UNAUTHORIZED` | 401 | 인증 실패 |
| `FORBIDDEN` | 403 | 권한 부족 |
| `NOT_FOUND` | 404 | requestId 없음 |
| `IDEMPOTENCY_CONFLICT` | 409 | 동일 requestId, 다른 내용 |
| `DUPLICATE_WITHDRAWAL` | 409 | 동일 withdrawalId 재시도 충돌 |
| `RATE_LIMITED` | 429 | 호출 제한 초과 |
| `SERVICE_UNAVAILABLE` | 503 | 일시적 장애 |

---

**7. 권장 조회 주기와 호출 제한**

> 내부 결정 필요. 현재 헥토 측 응답 시간 및 VTIM 처리 주기 기준으로 내부 검토 후 안내 드리겠습니다.
> 초안 기준: ACCEPTED/PROCESSING 상태 → 5~10초 주기, PENDING_CONFIRMATION 상태 → 30초 주기 권장 예정.

---

**8. 거래 보관 기간과 결과 조회 가능 기간**

> 내부 결정 필요. 현재 Vpay 서버의 거래 데이터는 별도 만료 정책 없이 영구 보관 중이나, 공식 정책을 확인 후 회신 드리겠습니다.

---

## 개발 전 내부 협의 필요 사항

1. EXPIRED 상태를 부릉에 PENDING_CONFIRMATION으로 노출할지, FAILED로 확정할지 정책 결정
2. 비동기 처리 방식 선택 (스레드풀 vs 배치 큐 vs DB polling)
3. 부릉 전용 API Key 발급 및 관리 방식
4. `accountMemo` → 헥토 `custAcntSumry` 매핑 시 바이트 제한 확인
5. 테스트용 헥토 mchtId/암호화 키 발급 여부 (헥토 측 문의 필요)
