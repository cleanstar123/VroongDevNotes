# 배달현황 크롤링 → 이관 → vr_delivery_list 저장 PowerShell 가이드

> 작성일: 2026-07-29  
> 브랜치: crawlingTest

---

## 전체 데이터 흐름

```
배민 외부 API
  → GET /api/v1/delivery/rider-delivery-history
  → rider_delivery_histories_if (IF 테이블, transfer_yn='N')
  → POST /api/v1/migration/migrate
  → vr_delivery_list (UPSERT, transfer_yn='Y')
```

---

## PowerShell 명령어

### 공통 변수 설정 (먼저 실행)

```powershell
$baseUrl = "https://back.vrplatformacct.com"
$headers = @{
    "X-API-Secret" = "local-dev-secret-key-2024"
    "Content-Type"  = "application/json"
}
```

---

### 1단계 — 크롤링 (배민 API → `rider_delivery_histories_if`)

배민 관리 API를 모든 센터(협력사)에 대해 병렬 호출하여 라이더별 배달건수를 IF 테이블에 저장한다.

```powershell
# fromDate ~ toDate 범위 데이터를 배민 API에서 조회 후 IF 테이블 적재
Invoke-RestMethod `
    -Method GET `
    -Uri "$baseUrl/api/v1/delivery/rider-delivery-history?fromDate=2026-08-05&toDate=2026-08-05" `
    -Headers $headers | ConvertTo-Json -Depth 10
```

- 결과: `rider_delivery_histories_if` 테이블에 `if_date=오늘날짜`, `transfer_yn='N'` 레코드 적재
- `fromDate` / `toDate` 형식: **`YYYY-MM-DD`** (하이픈 필수, `yyyyMMdd` 형식은 400 오류 발생)

---

### 2단계 — 이관 (`rider_delivery_histories_if` → `vr_delivery_list`)

IF 테이블의 미이관(`transfer_yn='N'`) 레코드를 100건 배치로 `vr_delivery_list`에 UPSERT한다.

```powershell
$body = @{ tableName = "rider_delivery_histories_if" } | ConvertTo-Json

Invoke-RestMethod `
    -Method POST `
    -Uri "$baseUrl/api/v1/migration/migrate" `
    -Headers $headers `
    -Body $body | ConvertTo-Json -Depth 10
```

- 결과: `vr_delivery_list`에 (SUPPLY_ID + RIDER_ID + DELIVERY_DATE) 기준 UPSERT
- 이관 완료된 레코드는 `transfer_yn='Y'`로 업데이트됨

---

## 참고: `vr_delivery_list` 주요 컬럼

| 컬럼 | 설명 |
|---|---|
| `SUPPLY_ID` | 협력사(센터) ID |
| `RIDER_ID` | 라이더 ID |
| `DELIVERY_DATE` | 배송일자 (yyyyMMdd) |
| `FINISH_CNT` | 배송 완료 건수 |
| `REJECT_CNT` | 거절 건수 |
| `ASSING_CANCEL_CNT` | 배차취소 건수 |
| `DELI_CANCEL_CNT` | 배달취소(라이더귀책) 건수 |
| `MOR_TIME_CNT` | 아침/점심 피크 건수 |
| `AFTER_NON_CNT` | 오후 논피크 건수 |
| `NIGHT_NON_CNT` | 저녁 논피크 건수 |
| `NIGHT_CNT` | 심야 피크 건수 |

---

## 주의 사항

- **순서 필수**: 1단계(크롤링) 완료 후 2단계(이관) 실행
- **API Secret**: 서버 환경변수 `VROONG_API_SECRET` 값을 `$headers`에 입력
- **포트**: 서버 기동 포트 확인 후 `$baseUrl` 수정 (기본값 `8080`)
- **실시간 데이터 크롤링**이 필요한 경우 아래 엔드포인트 사용 후 동일하게 이관

```powershell
# 실시간 데이터 크롤링 (rider_delivery_histories_now_if 저장)
Invoke-RestMethod -Method GET -Uri "$baseUrl/api/v1/delivery/status" -Headers $headers

# 이관 시 tableName 변경
$body = @{ tableName = "rider_delivery_histories_now_if" } | ConvertTo-Json
Invoke-RestMethod -Method POST -Uri "$baseUrl/api/v1/migration/migrate" -Headers $headers -Body $body
```

---

## 관련 주요 파일

| 역할 | 경로 |
|---|---|
| 크롤링 컨트롤러 | `api/.../controller/crawling/delivery/RiderDeliveryHistoryController.java` |
| 크롤링 유즈케이스 | `api/.../domain/crawling/baemin/delivery/usecase/DeliveryUseCase.java` |
| 배민 WebClient | `api/.../infrastructure/crawling/webclient/DeliveryWebClient.java` |
| 이관 컨트롤러 | `api/.../controller/crawling/migration/BaeminToVroongIfController.java` |
| 이관 유즈케이스 | `api/.../domain/crawling/baemin/migration/usecase/BaeminToVroongIfUseCase.java` |
| vr_delivery_list 레포 | `api/.../repository/vroong/VrDeliveryListQueryDslRepository.java` |
