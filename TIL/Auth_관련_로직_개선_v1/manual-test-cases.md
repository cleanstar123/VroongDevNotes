# Auth 수동 테스트 케이스 (2026-11-03, 수정본)

> **대상**: 묶음 1 (#7 #8), 묶음 2 (#6), 묶음 3 (#12 #13 #14 #19 #21 #27), 묶음 4 (#26) **환경**: dev (ECS) — **묶음 1~4 모두 dev에 배포한 상태에서 테스트** (TC-4, TC-11은 배포 전후 순서 주의) **도구**: 브라우저 2개(일반 + 시크릿), Postman, jwt.io, CloudWatch Logs, dev Redis·DB 접근

---

## 사전 준비

### dev ECS 임시 환경변수

|Spring property|환경변수|검증용 값|기본값|
|---|---|---|---|
|`jwt.access-token-validity`|`JWT_ACCESS_TOKEN_VALIDITY`|`60000` (1분)|3600000|
|`jwt.refresh-token-validity`|`JWT_REFRESH_TOKEN_VALIDITY`|`300000` (5분)|86400000|

- dev를 쓰는 다른 팀원에게 테스트 시간 동안 토큰이 1분마다 만료된다고 미리 공유
- 테스트 완료 후 두 환경변수 제거

### 테스트 계정

|계정|userType|사용 TC|
|---|---|---|
|포털 본사 계정|C|TC-1, 2, 3, 6, 8, 9, 11|
|포털 협력사 계정|S|TC-4(S), TC-5|
|포털 관리자 계정|A|TC-1 반복 (선택)|
|모바일 라이더 계정|R|TC-7, TC-10, TC-12|
|모바일 관리자 계정|—|TC-12, TC-13(#12)|

### 공통 팁

- localStorage 토큰 값은 JSON 문자열이라 앞뒤에 `"`가 붙어 있음 → jwt.io·Postman에 붙일 때 따옴표 제거
- refresh 토큰 직접 호출: `POST /api/v1/auth/refresh`, Body `{ "refreshToken": "<값>" }`. X-API-Secret 불필요 (화이트리스트)
- Redis 세션 확인: 포털 세션 키에서 해당 userId의 sessionId·TTL 확인

---

## 묶음 1 — refresh (#7 #8)

### TC-1. 정상 refresh rotation

1. 포털 웹에서 본사(C) 계정 로그인
2. DevTools → Local Storage: `refreshToken` 키 존재, `sessionId` 키 없음 확인 → 값 복사 (A)
3. jwt.io로 A 디코딩 → `sessionId`, `exp` 기록
4. 1분 이상 대기 → 화면에서 목록 조회 등 동작

**예상 결과**

- [x] 로그아웃 없이 정상 동작 ✅ 2026-11-03
- [x] Network 탭: API 401 → `/api/v1/auth/refresh` 200 → 원래 API 재시도 200 ✅ 2026-11-03
- [x] `refreshToken` 값이 A와 다름 (rotation) ✅ 2026-11-03
- [x] 새 refresh 토큰의 `sessionId` = A의 `sessionId`, `exp` = A의 `exp` (연장 없음) ✅ 2026-11-03
- [x] 새 Access 토큰에 `userName`, `userType=C`, 같은 `sessionId` 포함 ✅ 2026-11-03
- [x] 로그: `Access Token 갱신 완료: userId=...` ✅ 2026-11-03
- [x] Redis 세션 TTL이 로그인 시점 기준 약 5분 (1분이 아님) ✅ 2026-11-03

### TC-2. 절대 만료

1. 본사(C) 로그인 → 중간에 1~2번 refresh가 일어나도록 화면 사용
2. 로그인 후 5분 경과 → 화면 동작

**예상 결과**

- [ ] 로그인 화면으로 이동
- [ ] `/api/v1/auth/refresh` 401
- [ ] 중간 refresh가 있었어도 로그인 후 5분에 만료됨 (rotation으로 연장되지 않음)

### TC-3. 다른 기기 로그인 — refresh 엔드포인트 세션 검증 (#7)

> 화면만으로는 #7이 검증되지 않음. B 로그인 순간 A의 Access 토큰은 PartnerPortalJwtFilter(기존 기능)에서 먼저 차단되므로 refresh를 직접 호출해야 함.

1. 브라우저 A: 본사(C) 로그인 → Local Storage의 refresh 토큰 복사 (A-refresh)
2. 브라우저 B: 같은 계정 로그인
3. Postman: A-refresh로 `/api/v1/auth/refresh` 호출
4. 브라우저 A에서 화면 동작

**예상 결과**

- [ ] 3번: 401, `errorCode: SESSION_INVALIDATED`
- [ ] 3번 로그: `세션 무효화됨 (다른 기기에서 로그인): userId=..., sessionId=...`
- [ ] 4번: "다른 기기에서 로그인" 안내 후 로그인 화면
- [ ] 브라우저 B는 계속 정상 동작
- [ ] Redis 세션이 B의 sessionId로 유지됨 (3번 호출이 덮어쓰지 않음)

### TC-4. 구 토큰 전환 (배포 전 발급 토큰)

> **묶음 1 배포 전**에 준비해야 함

**준비 (구 버전 dev에서)**

1. 본사(C) 로그인 → refresh 토큰 저장 (old-C)
2. 협력사(S) 로그인 → refresh 토큰 저장 (old-S)
3. jwt.io: 둘 다 `sessionId` 없음 확인 (old-S에는 `suppId` 있음)

**묶음 1 배포 후** 4. Postman: old-C로 refresh 호출 5. Postman: old-S로 refresh 호출 6. 5번에서 받은 새 refresh 토큰으로 다시 refresh 호출

**예상 결과**

- [ ] 4, 5번: 200, 새 Access·Refresh 토큰 발급
- [ ] 로그: `구 형식의 refresh 토큰 수신 (sessionId 없음, 전환 기간): userId=...` (각 1회)
- [ ] 새 토큰에 `sessionId` 생겼음, Redis에 같은 sessionId 저장됨
- [ ] 새 refresh 토큰 `exp` = 구 토큰의 `exp` (그대로 유지)
- [ ] 5번의 새 Access 토큰에 `suppId` 있음
- [ ] 6번: 200, 구 토큰 로그 없음 (신규 경로로 처리)

### TC-5. SUPPLIER refresh 후 수수료 수정 (#8, SUPPLIER suppId 유지)

1. 협력사(S) 로그인
2. 1분 이상 대기 → **수수료가 아닌 다른 포털 화면**에서 동작해 refresh 발생 확인 (Network 탭에서 `/auth/refresh` 200)
3. 라이더 수수료 수정 화면에서 수정·저장
4. 새 Access 토큰 jwt.io 디코딩

**예상 결과**

- [ ] 3번: 수정 성공 (403·권한 오류 없음)
- [ ] 4번: `suppId` = userId, `userType=S`, `sessionId` 있음
- [ ] 추가 확인: refresh 없이 Access 만료 상태에서 바로 수수료 수정을 누르면 어떻게 되는지 기록 (RiderFeeController가 401을 반환해 refresh를 유도하는지)

### TC-6. 비활성 계정 refresh 차단

1. 본사(C) 로그인 → refresh 토큰 복사
2. dev DB: 해당 계정 `use_yn='N'`
3. Postman: refresh 호출 (또는 1분 대기 후 화면 동작)

**예상 결과**

- [ ] 401, 토큰 미발급
- [ ] 로그: `비활성화된 계정으로 refresh 시도: userId=...`
- [ ] 화면 사용 시 로그인 화면 이동

**사후**: `use_yn='Y'` 복원

### TC-7. 라이더(R) 토큰으로 포털 refresh 차단

1. 라이더 계정으로 `POST /api/v1/mob/login` → 응답의 refresh 토큰 확보
2. Postman: 그 토큰으로 `/api/v1/auth/refresh` 호출

**예상 결과**

- [ ] 401, 토큰 미발급
- [ ] 로그: `포털 미허용 userType으로 refresh 시도` 또는 사용자 조회 실패 로그 (라이더가 포털 사용자 테이블에 없을 수 있음 — 어느 쪽이든 401이면 통과, 실제 로그 문구 기록)
- [ ] **Redis에 해당 라이더 userId로 포털 세션이 생기지 않음**

### TC-8. Access 토큰으로 refresh 시도 차단

1. 본사(C) 로그인 → Network 탭 Authorization 헤더에서 Access 토큰 복사
2. Postman: Body의 `refreshToken`에 Access 토큰을 넣어 `/api/v1/auth/refresh` 호출

**예상 결과**

- [ ] 401, 토큰 미발급

---

## 묶음 2 — #6 MobileJwtFilter ACCESS 체크

### TC-10. 모바일 API에 refresh 토큰 사용 차단

1. 라이더 `POST /api/v1/mob/login` → Access·Refresh 토큰 확보
2. Postman: 라이더용 조회 API(`/api/v1/mob/...`)에 `Authorization: Bearer <Access>` + 앱과 같은 `X-API-Secret` → 정상 응답 확인
3. 같은 요청을 `Authorization: Bearer <Refresh>`로 호출

**예상 결과**

- [ ] 2번: 200
- [ ] 3번: 401, 응답 형식이 기존과 동일 (`{"success":false,"error":"...","errorCode":"UNAUTHORIZED"}`)
- [ ] 로그: `잘못된 토큰 타입: REFRESH (expected: ACCESS)`

---

## 묶음 4 — #26 프론트 저장 키

### TC-11. 구 저장 키 사용자 전환

> **묶음 4 배포 전**에 준비

1. 구 프론트에서 본사(C) 로그인 → Local Storage에 `sessionId` 키(refresh 토큰) 있음 확인
2. 브라우저를 닫지 않고 묶음 4 배포
3. 새로고침 없이 또는 새로고침 후 1분 이상 대기 → 화면 동작

**예상 결과**

- [ ] 로그아웃 없이 refresh 성공 (구 키 폴백)
- [ ] refresh 후 `refreshToken` 키 생성, `sessionId` 키 삭제
- [ ] 다음 refresh도 정상 (이번에는 `refreshToken` 키 사용)
- [ ] 로그아웃 시 `refreshToken` 키 삭제

---

## 회귀 테스트 — 기존 기능

### TC-12. 모바일 앱 기본 흐름 (#6, #19 영향)

- [ ] 라이더 앱 로그인 → 홈 진입 → 주요 화면 2~3개 조회
- [ ] 모바일 관리자 로그인 → 관리자 홈 → 협력사/라이더 전환(switch) 동작
- [ ] Access 만료(1분) 후 앱 동작 → 기존처럼 로그인 화면 이동 (앱은 refresh 미사용, 동작 변화 없어야 함)

### TC-13. 포털 기본 흐름 (#19)

- [ ] 본사·협력사·관리자 각각 로그인 → 주요 화면 조회
- [ ] 외부 출금 API(`/api/v1/withdrawals/**`)를 dev에서 쓰고 있다면 X-API-Key로 1회 호출해 정상 응답 확인

---

## 묶음 3 — 메시지·로그

### TC-14. 로그인 실패 메시지 (#27)

|시나리오|기대|
|---|---|
|포털: 없는 아이디|"아이디 또는 비밀번호를 확인해주세요"|
|포털: 비밀번호 틀림|위와 동일 (두 경우 구분 불가해야 함)|
|포털: 비활성 계정|"비활성화된 계정입니다" 유지 (의도된 유지)|
|앱 라이더: 없는 아이디 / 비밀번호 틀림|기존과 같은 안내 UI|

### TC-15. 민감정보 로그 제거

|항목|재현|확인|
|---|---|---|
|#12|모바일 관리자 로그인|`Request - userId: ...`만 출력, password 없음|
|#13 및 추가 7곳|라이더 회원가입 이메일 요청 → 인증 → 가입 완료 (실패 케이스 1회 포함)|6자리 코드가 어떤 로그에도 없음 (CloudWatch에서 실제 받은 코드 값으로 검색 → 0건)|
|#14|포털 비밀번호 찾기 → 재설정|`token=` 뒤에 앞 8자 + `***`만 출력|
|#21|아무 보호 API에 틀린 X-API-Secret으로 호출|`(length=N)`만 출력|

---

## 운영 배포 직후 모니터링 (묶음 1 기준, 30분~1시간)

|로그|기대|이상 시|
|---|---|---|
|`API Secret 헤더 누락: POST /api/v1/auth/refresh`|0건|화이트리스트 미적용 → 배포 확인|
|`Access Token 갱신 완료: userId=`|증가|0건 지속 시 refresh 미동작|
|`구 형식의 refresh 토큰 수신`|배포 직후 발생, 24시간 후 0건|24시간 후 지속 시 출처 확인|
|`세션 검증 중 오류`|0건|Redis 상태 확인|
|`포털 미허용 userType` / `비활성화된 계정으로 refresh`|소수|급증 시 원인 확인|

**롤백 기준**: 포털 로그아웃·권한 오류 문의 증가, 협력사 수수료 수정 오류, `세션 검증 중 오류` 지속 → 묶음 1 전체 롤백

---

## 테스트 완료 후 정리

- [ ] dev ECS 환경변수 `JWT_ACCESS_TOKEN_VALIDITY`, `JWT_REFRESH_TOKEN_VALIDITY` 제거 후 재배포
- [ ] TC-6 계정 `use_yn='Y'` 복원
- [ ] TC-5 "refresh 없이 바로 수수료 수정" 결과 기록 (후속 이슈 여부 판단)
- [ ] TC-7 실제 로그 문구 기록