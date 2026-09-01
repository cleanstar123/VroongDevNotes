# 부릉(VROONG) 배치 API 실행 가이드

배민 인증부터 라이더/협력사 데이터 크롤링, DB 이관까지의 PowerShell 실행 스크립트 모음입니다.

## 목차

1. [환경 정보](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#%ED%99%98%EA%B2%BD-%EC%A0%95%EB%B3%B4)
2. [인증](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#1-%EC%9D%B8%EC%A6%9D)
3. [배달현황](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#2-%EB%B0%B0%EB%8B%AC%ED%98%84%ED%99%A9)
4. [협력사 배달비](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#3-%ED%98%91%EB%A0%A5%EC%82%AC-%EB%B0%B0%EB%8B%AC%EB%B9%84)
5. [라이더 배송 이력](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#4-%EB%9D%BC%EC%9D%B4%EB%8D%94-%EB%B0%B0%EC%86%A1-%EC%9D%B4%EB%A0%A5)
6. [라이더 정보](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#5-%EB%9D%BC%EC%9D%B4%EB%8D%94-%EC%A0%95%EB%B3%B4)
7. [배송 일일 이력](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#6-%EB%B0%B0%EC%86%A1-%EC%9D%BC%EC%9D%BC-%EC%9D%B4%EB%A0%A5-deliverydailyhistoriesjob)
8. [파트너 상세 정보](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#7-%ED%8C%8C%ED%8A%B8%EB%84%88-%EC%83%81%EC%84%B8-%EC%A0%95%EB%B3%B4)
9. [선정산](https://claude.ai/chat/bcc9b9cd-9ea6-4e30-97e1-0b6230e469ba#8-%EC%84%A0%EC%A0%95%EC%82%B0)

---

## 환경 정보

|구분|값|
|---|---|
|로컬 API 서버|`http://localhost:8080`|
|로컬 SMS/릴레이 웹훅 서버|`http://localhost:8090`|
|개발 ALB|`http://vroong-alb-1049542169.ap-northeast-2.elb.amazonaws.com:8080`|
|인증 헤더|`X-API-SECRET: local-dev-secret-key-2024` (개발 전용)|

> 대부분의 API 호출은 이 `X-API-SECRET` 헤더가 필요합니다. 아래 예시에서는 반복을 줄이기 위해 최초 1회만 전체 헤더를 표기하고, 이후엔 동일하게 적용된다고 가정합니다.

---

## 1. 인증

### 1-1. SMS 인증코드 웹훅 — 운영

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8090/webhook/sms" `
    -ContentType "application/json" `
    -Body '{"message":"954864"}'
```

### 1-2. SMS 인증코드 웹훅 — 개발

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://vroong-alb-1049542169.ap-northeast-2.elb.amazonaws.com:8080/api/v1/baemin/auth/sms-webhook" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -ContentType "application/json" `
    -Body '{"message":"681469"}'
```

### 1-3. 현재 세션 등록

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8090/relay/auth/register-current-session" `
    -ContentType "application/json" `
    -Body "{}"
```

### 1-4. 릴레이 센터 목록 조회

쿼리 파라미터 방식:

```powershell
Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8090/relay/center/list?sessionId=c7e7cca5-aafb-41f6-a764-38c875ac6e83"
```

헤더 방식:

```powershell
Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8090/relay/center/list" `
    -Headers @{ "x-session-id" = "c7e7cca5-aafb-41f6-a764-38c875ac6e83" }
```

---

## 2. 배달현황

각 도메인은 **조회 → 이관** 순서로 구성됩니다.

### 2-1. 조회

```powershell
Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/delivery/status" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
```

### 2-2. 이관 → `rider_delivery_histories_now_if`

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "rider_delivery_histories_now_if"
    }'
```

---

## 3. 협력사 배달비

### 3-1. 조회

```powershell
Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/partner/delivery-fee?fromDate=2026-06-14&toDate=2026-06-14" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
```

### 3-2. 이관 → `partner_delivery_fees_if`

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "partner_delivery_fees_if"
    }'
```

---

## 4. 라이더 배송 이력

### 4-1. 조회

```powershell
Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/delivery/rider-delivery-history?fromDate=2026-06-14&toDate=2026-06-14" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
```

### 4-2. 이관 → `rider_delivery_histories_if`

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{"tableName":"rider_delivery_histories_if","targetDate":"20260614"}'
```

---

## 5. 라이더 정보

### 5-1. 기본 정보 크롤링 → `rider_info_if`

```powershell
Write-Host "라이더 기본 정보 크롤링 시작..."
$riderInfoResult = Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/rider/info" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
Write-Host "라이더 기본 정보 크롤링 완료:" ($riderInfoResult | ConvertTo-Json)
```

### 5-2. 상세 정보 크롤링 → `rider_detail_info_if`

```powershell
Write-Host "라이더 상세 정보 크롤링 시작..."
$riderDetailResult = Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/rider/info/detail" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
Write-Host "라이더 상세 정보 크롤링 완료:" ($riderDetailResult | ConvertTo-Json)
```

### 5-3. 이관 → `rider_info_if` → `vr_rider_info`

```powershell
Write-Host "라이더 정보 이관 시작..."
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "rider_info_if"
    }'
```

---

## 6. 배송 일일 이력 (`deliveryDailyHistoriesJob`)

### 6-1. 크롤링

```powershell
Write-Host "배송 일일 이력 크롤링 시작..."
$deliveryDailyResult = Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/delivery/daily-delivery-status?fromDate=2026-08-31&toDate=2026-08-31" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
Write-Host "배송 일일 이력 크롤링 완료:" ($deliveryDailyResult | ConvertTo-Json -Depth 5)
```

### 6-2. 이관 → `delivery_daily_histories_if`

```powershell
Write-Host "배송 일일 이력 이관 시작..."
$deliveryDailyMigrateResult = Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "delivery_daily_histories_if"
    }'
Write-Host "배송 일일 이력 이관 완료:" ($deliveryDailyMigrateResult | ConvertTo-Json)
```

### 6-3. 라이더 배송 이력 크롤링 (`riderDeliveryHistoriesJob`)

```powershell
Write-Host "라이더 배송 이력 크롤링 시작..."
$riderDeliveryResult = Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/delivery/rider-delivery-history?fromDate=2026-08-31&toDate=2026-08-31" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
Write-Host "라이더 배송 이력 크롤링 완료:" ($riderDeliveryResult | ConvertTo-Json -Depth 5)
```

### 6-4. 라이더 배송 이력 이관 → `rider_delivery_histories_if`

```powershell
Write-Host "라이더 배송 이력 이관 시작..."
$riderDeliveryMigrateResult = Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "rider_delivery_histories_if"
    }'
Write-Host "라이더 배송 이력 이관 완료:" ($riderDeliveryMigrateResult | ConvertTo-Json)
```

---

## 7. 파트너 상세 정보

### 7-1. 크롤링 → `partner_details_if`

```powershell
Write-Host "파트너 상세 정보 크롤링 시작..."
$partnerDetailResult = Invoke-RestMethod `
    -Method GET `
    -Uri "http://localhost:8080/api/v1/partner/detail" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" }
Write-Host "파트너 상세 정보 크롤링 완료:" ($partnerDetailResult | ConvertTo-Json -Depth 5)
```

### 7-2. 이관 → `partner_details_if` → `vr_supplier_info`

```powershell
Write-Host "파트너 상세 정보 이관 시작..."
$partnerMigrateResult = Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "partner_details_if"
    }'
Write-Host "파트너 상세 정보 이관 완료:" ($partnerMigrateResult | ConvertTo-Json)
```

---

## 8. 선정산

### 8-1. 이관 → `vr_rider_acct_daily`

```powershell
Invoke-RestMethod `
    -Method POST `
    -Uri "http://localhost:8080/api/v1/migration/migrate" `
    -ContentType "application/json" `
    -Headers @{ "X-API-SECRET" = "local-dev-secret-key-2024" } `
    -Body '{
        "tableName": "vr_rider_acct_daily",
        "targetDate": "20260420"
    }'
```

---

## 참고 사항

- 이관(`/api/v1/migration/migrate`) 호출은 `tableName`만 필수이며, `targetDate`를 생략하면 기본적으로 당일(또는 최신 크롤링분) 데이터를 대상으로 처리되는 것으로 보입니다.
- 특정 일자를 지정하려는 경우 조회 API는 `fromDate`/`toDate` 쿼리 파라미터를, 이관 API는 `targetDate` 바디 필드를 사용합니다.
- 크롤링(`GET`) → 이관(`POST /migration/migrate`)의 2단계 흐름이 기본 패턴이며, 크롤링 결과가 임시 테이블(`*_if`)에 적재된 뒤 이관 API를 통해 실제 서비스 테이블(`vr_*`)로 옮겨집니다.
- `X-API-SECRET` 값은 개발 환경 전용이며, 운영 환경에서는 별도 값을 사용해야 합니다.