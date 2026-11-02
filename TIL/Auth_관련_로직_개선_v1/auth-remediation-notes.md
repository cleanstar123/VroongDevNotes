# 인증 수정 작업 정리 (2026-11-02, 번호 정정본)

> 항목 번호는 최초 분석 보고서의 "발견된 문제점 목록"(#1~#27)과 이후 신규 발견 항목(#28~)을 기준으로 한다. 경로·라인은 Claude Code 보고 기준이므로 커밋 전에 `git diff`로 한 번 더 확인할 것.

---

## [1] 변경 파일 전체 목록

### Backend (`C:\wonjun\vroong-backend`, 기준 경로 `api/src/main/java/com/vroong/api/`)

|파일|항목|변경 내용|
|---|---|---|
|`app/security/ApiSecretFilter.java`|#8|`/api/v1/auth/refresh` 화이트리스트 추가 (+1줄)|
||#21|API Secret 불일치 로그: 앞 10자 노출 → `(length=N)`|
|`app/security/MobileJwtFilter.java`|#6|ACCESS 토큰 타입 체크 추가 (응답 형식은 기존 유지)|
|`app/security/PartnerPortalJwtFilter.java`|#19|`@Order(-99)` → `@Order(-97)`|
|`infrastructure/security/jwt/JwtTokenProvider.java`|#8|`createRefreshToken(userId, sessionId)`, `createRefreshToken(userId, sessionId, Date)`, `getRefreshTokenTtlSeconds()` 추가|
|`app/controller/crawling/auth/AuthController.java`|#7, #8|`refreshToken` 전면 재작성<br>- 검증 순서: 서명/만료 → tokenType=REFRESH → DB 재조회(실패 시 401) → userType 허용(S/C/A) → 계정 상태(useYn) → Redis 검증 또는 구 토큰 saveSession → 발급<br>- SUPPLIER: `createMobileAccessToken(userId, userId, "S", sessionId)` (suppId 유지)<br>- 그 외: `createAccessToken(userId, userType, userName, sessionId)`<br>- 새 refresh 토큰 exp = 기존 exp 유지 (로그인 후 24시간 절대 만료)<br>- Redis 예외 시 401|
|`app/controller/portal/partner/PartnerPortalAuthController.java`|#8|SUPPLIER: `createMobileRefreshToken` → `createRefreshToken(userId, sessionId)`<br>HQ: `createRefreshToken(userId)` → `createRefreshToken(userId, sessionId)`<br>Redis 세션 TTL: Access TTL → Refresh TTL|
||#14|비밀번호 재설정 token 로그 → 앞 8자 마스킹|
|`domain/portal/partner/usecase/PartnerPortalAuthUseCase.java`|#27|포털 로그인 실패 메시지 → "아이디 또는 비밀번호를 확인해주세요"|
|`domain/mobile/usecase/MobLoginUseCase.java`|#27|라이더 로그인 실패 메시지 동일하게 통일 (앱 `useLogin.ts:122`의 `includes('비밀번호')` 분기로 UI 동일 확인)|
|`app/controller/mobile/MobAdminLoginController.java`|#12|`Request 객체: {}` → `userId`만 로그|
|`domain/mobile/usecase/MobRiderRegistrationUseCase.java`|#13 + 추가 로그|OTP(6자리) 코드 로그 제거 — 기존 #13 1곳 + 추가 발견 6곳|
|`app/controller/mobile/MobLoginController.java`|추가 로그|회원가입 토큰(OTP) 로그 제거 4곳|

**신규 파일 (untracked)**

|파일|항목|내용|
|---|---|---|
|`api/src/test/groovy/.../crawling/auth/AuthControllerRefreshTokenSpec.groovy`|#7, #8|11개 시나리오|

### Frontend (`C:\wonjun\vroong-front`)

|파일|항목|변경 내용|
|---|---|---|
|`src/shared/utils/storage.ts`|#26|`STORAGE_KEYS.REFRESH_TOKEN` 추가|
|`src/shared/api/interceptors.ts`|#26|`REFRESH_TOKEN` 우선 읽기 + `SESSION_ID` 폴백, rotation 후 `REFRESH_TOKEN` 저장 및 `SESSION_ID` 삭제|
|`src/features/auth/hooks/useAuth.ts`|#26|로그인 시 `REFRESH_TOKEN` 저장, 로그아웃 시 삭제|

---

## [2] 커밋 / 배포 묶음

### 배포 묶음 1 — #7, #8 refresh 수정 (반드시 함께 배포)

화이트리스트만 먼저 배포되면 기존 refresh 코드가 열려 SUPPLIER suppId 누락 버그(수수료 수정 불가)가 드러난다. 아래 커밋은 하나의 릴리즈로 배포한다. 롤백도 묶음 전체를 한 번에 되돌린다.

|커밋|메시지|파일|
|---|---|---|
|1-A|`feat(jwt): add sessionId-aware refresh token creation and refresh TTL accessor (#8)`|`JwtTokenProvider.java`|
|1-B|`fix(portal-auth): include sessionId in refresh token and use refresh TTL for Redis session (#8)`|`PartnerPortalAuthController.java` — #8 hunk만 (`git add -p`)|
|1-C|`fix(auth): rewrite token refresh with session validation, absolute expiry and DB re-query (#7 #8)`|`AuthController.java` 전체 + `ApiSecretFilter.java` 화이트리스트 hunk만 (`git add -p`)|
|1-D|`test(auth): add refresh token rotation spec (#7 #8)`|`AuthControllerRefreshTokenSpec.groovy`|

### 배포 묶음 2 — #6 (독립 배포 가능)

|커밋|메시지|파일|
|---|---|---|
|2-A|`fix(security): reject non-ACCESS tokens in MobileJwtFilter (#6)`|`MobileJwtFilter.java`|

### 배포 묶음 3 — 로그·순서·메시지 (독립 배포 가능)

|커밋|메시지|파일|
|---|---|---|
|3-A|`fix(security): resolve filter @Order collision (#19)`|`PartnerPortalJwtFilter.java`|
|3-B|`fix(auth): unify login failure messages (#27)`|`PartnerPortalAuthUseCase.java`, `MobLoginUseCase.java`|
|3-C|`fix(log): remove passwords, OTP codes and secrets from logs (#12 #13 #14 #21)`|`MobAdminLoginController.java`, `MobRiderRegistrationUseCase.java`, `MobLoginController.java`, `PartnerPortalAuthController.java` 마스킹 hunk, `ApiSecretFilter.java` 마스킹 hunk|

`git add -p` 분리 대상:

- `PartnerPortalAuthController.java`: 토큰 생성·TTL hunk → 1-B / reset token 마스킹 hunk → 3-C
- `ApiSecretFilter.java`: `/api/v1/auth/refresh` 화이트리스트 hunk → 1-C / `(length=N)` 로그 hunk → 3-C

### 배포 묶음 4 — #26 프론트 (묶음 1 배포 후)

|커밋|메시지|파일|
|---|---|---|
|4-A|`fix(auth): store refresh token under REFRESH_TOKEN key with legacy fallback (#26)`|`storage.ts`, `interceptors.ts`, `useAuth.ts`|

### 권장 배포 순서

1. 묶음 3, 묶음 2 (위험 낮음, 먼저 배포 가능)
2. 묶음 1 (백엔드) — dev 검증 완료 후
3. 묶음 4 (프론트) — 묶음 1 운영 확인 후

---

## [3] 테스트

`AuthControllerRefreshTokenSpec` 11개 시나리오. 요청 대비 상태:

|시나리오|상태|
|---|---|
|sessionId 있는 정상 토큰 → 같은 sessionId, exp 유지|✅|
|Redis 세션 불일치 → SESSION_INVALIDATED|✅|
|sessionId 없는 구 토큰 → 새 sessionId + saveSession|✅|
|만료 refresh 토큰 → 401|✅|
|ACCESS 토큰으로 refresh → 401|✅|
|Redis 예외 → 401|✅|
|SUPPLIER → `createMobileAccessToken`, suppId 포함|✅|
|HQ → `createAccessToken`, userName 포함|⚠️ 기존 C 타입 정상 케이스로 대체 검증|
|DB 예외 / 사용자 없음 → 401|✅|
|비활성 계정 → 401|✅|
|비허용 userType(R) → 401, saveSession 미호출|✅|
|RiderFeeController 수정 권한 (refresh 후 SUPPLIER)|❌ 미작성 → dev 수동 검증으로 대체|

실행: `gradlew.bat :api:test` (전체) — 로컬 Gradle Worker 오류 해결 필요.

---

## [4] 잔여 작업 전체 목록

### 즉시 (코드 외 운영 조치)

|#|내용|다음 액션|
|---|---|---|
|#3|`application-local.yml`, `application-dev.yml`에 RDS 비밀번호·Gmail 앱 비밀번호·Hecto 키 평문|**자격증명 로테이션 먼저** (git 히스토리에 남아 있음). 이후 env var 체계로 이관|

### 다음 작업 후보 (B — 사전 작업 후 수정)

|순위|#|내용|선행 조건|다음 액션|정책 판단|
|---|---|---|---|---|---|
|1|#4|ApiSecretFilter: 임의 `Bearer xxx`로 API Secret 우회|설계 완료|Bearer 분기에서 `서명 유효 && tokenType=ACCESS`일 때만 통과. 웹은 X-API-Secret을 보내지 않으므로 secret 추가 방식은 사용하지 않음|불필요|
|1|#24|API Secret 화이트리스트 `/partner-portal/auth/**` 전체 → `/user/{userId}`도 secret 없이 호출 가능|#4와 함께|공개 7개 경로만 명시. `/user/{userId}`는 웹이 Bearer로 호출하므로 #4 적용 후 안전|불필요|
|2|#1|JWT 시크릿이 코드 기본값 (yml에 `jwt.secret` 없음)|운영·dev ECS에 **현재와 같은 값**으로 `JWT_SECRET` 설정|설정 확인 후 코드 기본값 제거 (값 변경 시 전원 로그아웃)|불필요|
|2|#2-B|AES 키 하드코딩|ECS에 **현재와 같은 값** 설정|환경변수로 분리 (키 값 변경은 #2-C)|불필요|
|3|#5|CORS `*` + credentials 허용|허용 Origin 확정: `https://back.vrplatformacct.com`, `https://mob.vrplatformacct.com`, `https://www.vrplatformacct.com`, dev `http://localhost:8080` (앱 Capacitor origin 포함 여부 확인 필요)|환경별 화이트리스트 + `vroong.api.cors.allowed-origins`를 실제로 읽도록 수정|불필요|
|4|#11|로그아웃 API 없음 (Redis 세션 미삭제)|—|백엔드 엔드포인트 추가 후 웹 로그아웃에서 호출|불필요|
|5|#15, #16|resetPassword 비동기 체인 비원자성 / 비밀번호 변경 후 세션 미무효화|—|동기 메서드로 분리 후 트랜잭션 + `deleteSession`|불필요|
|6|#22|switch-to-supplier/rider 응답에 BCrypt 해시·암호화 PII 포함|앱 `HomePageForAdmin.tsx`가 실제 호출 중|앱이 쓰는 응답 필드 확인 후 Response DTO 분리|불필요|
|7|#28|ApiSecretFilter `@Value` 키(`vroong.api.secret.enabled`) ↔ yml(`secret-config.enabled`) 불일치|운영·dev ECS에 `API_SECRET_ENABLED` / `VROONG_API_SECRET_ENABLED` 설정 여부 확인 (설정돼 있으면 키 수정 시 필터가 꺼질 수 있음)|확인 후 키 일치|불필요|
|8|#17|PartnerPortalJwtFilter sessionId 없으면 suppId 폴백|묶음 1 배포 후 24시간 경과 (sessionId 없는 Access 토큰 소멸)|폴백 제거 → B로 전환|불필요|

### 보류 (C — 이번 범위 아님)

|#|내용|보류 사유|
|---|---|---|
|#2-C|AES 키 값 변경 + 기존 암호문 재암호화|데이터 마이그레이션 필요|
|#9|모바일 refresh 시 포털용 토큰으로 교체|앱이 refresh 미사용. 이제 refresh는 S/C/A만 허용|
|#10|모바일 로그인 Redis 세션 미저장 (중복 로그인 방지 미동작)|정책 결정 필요|
|#18|포털/모바일 ID 대소문자 정규화 차이|의도된 설계로 확인|
|#20|OTP 시도 횟수 제한 없음|Rate limit 정책·인프라 필요|
|#23|MobileJwtFilter + MobInterceptor JWT 중복 검증|attribute 키 차이로 NPE 위험, 분석 필요|
|#25|Refresh 토큰 블랙리스트 없음|Redis 설계 필요 (현재 절대 만료 24시간으로 완화)|

### 기타 기록

- `/api/v1/auth/refresh`가 배민 크롤링용 `AuthController` 안에 있음 → 별도 컨트롤러 분리 권장 (낮음)
- 모바일 관리자 로그인 refresh 토큰(sessionId 없음, userType S/C/A)도 `/auth/refresh` 구 토큰 분기를 통과할 수 있음 → 구 토큰 WARN이 24시간 후에도 남으면 출처 확인
- 삭제 후보 엔드포인트(최초 보고서 5절)는 이번 작업에서 손대지 않음

---

## [5] dev 검증 가이드

### 환경변수 (dev ECS)

|Spring property|환경변수|검증용 값|
|---|---|---|
|`jwt.access-token-validity`|`JWT_ACCESS_TOKEN_VALIDITY`|`60000` (1분)|
|`jwt.refresh-token-validity`|`JWT_REFRESH_TOKEN_VALIDITY`|`300000` (5분)|

검증 후 환경변수를 제거해 기본값(1시간 / 24시간)으로 원복할 것.

### 시나리오

가능하면 Postman 직접 호출보다 **웹 화면에서 실제로 사용하면서** 확인한다 (프론트 interceptor 동작까지 함께 검증됨).

1. **정상 rotation** — 로그인 → 1분 이상 대기 → 화면 조작 → 로그아웃되지 않고 계속 사용 가능
    - 로그: `Access Token 갱신 완료: userId=...`
    - DevTools: localStorage에 `refreshToken` 존재, `sessionId` 키 삭제 (묶음 4 적용 시)
    - 새 refresh 토큰 exp = 최초 로그인 + 5분
2. **절대 만료** — 로그인 후 5분 경과 → 화면 조작 → 로그인 화면 이동
3. **다른 기기 로그인 (#7)** — 브라우저 A 로그인 → 브라우저 B 같은 계정 로그인 → A에서 1분 후 조작 → "다른 기기에서 로그인" 안내 후 로그인 화면
    - 로그: `세션 무효화됨 (다른 기기에서 로그인)`
4. **SUPPLIER refresh 후 수수료 수정** — SUPPLIER 로그인 → 1분 이상 대기(refresh 발생) → 협력사 라이더 수수료 수정 화면에서 수정 → 성공해야 함
    - 대상 API: `RiderFeeController` (`/api/v1/supplier-rider-fees` 계열 — 정확한 경로 확인 필요)
    - 새 Access 토큰을 jwt.io 등으로 디코딩해 `suppId` 클레임 존재 확인
5. **비활성 계정** — dev DB에서 `use_yn='N'` 변경 → refresh 시도 → 로그인 화면 이동
    - 로그: `비활성화된 계정으로 refresh 시도` / 테스트 후 `use_yn='Y'` 복원

### 운영 배포 직후 모니터링

|로그|기대|
|---|---|
|`API Secret 헤더 누락: POST /api/v1/auth/refresh`|묶음 1 배포 후 0건|
|`Access Token 갱신 완료: userId=`|새로 발생하기 시작|
|`구 형식의 refresh 토큰 수신 (sessionId 없음, 전환 기간)`|배포 직후 발생, 24시간 내 0건으로 수렴|
|`세션 검증 중 오류`|0건 (발생 시 Redis 상태 확인)|
|`포털 미허용 userType으로 refresh 시도` / `비활성화된 계정으로 refresh 시도`|소수 — 급증 시 원인 확인|

롤백 판단 기준: 배포 후 포털 사용자 로그아웃 문의 증가, 또는 `세션 검증 중 오류` 지속 발생 → 묶음 1 전체 롤백.