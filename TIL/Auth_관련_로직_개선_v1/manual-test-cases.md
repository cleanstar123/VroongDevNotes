# Auth 수동 테스트 케이스 (2026-11-02)

> **대상 배포 묶음**: 묶음 1 (#7 #8), 묶음 2 (#6), 묶음 3 (#12 #13 #14 #19 #21 #27)
> **테스트 환경**: dev (ECS)
> **사전 조건**: 아래 환경변수를 dev ECS에 임시 적용 후 테스트, 완료 후 제거

---

## 사전 준비

### dev ECS 임시 환경변수 설정

| Spring property | 환경변수 | 검증용 값 | 기본값 |
|---|---|---|---|
| `jwt.access-token-validity` | `JWT_ACCESS_TOKEN_VALIDITY` | `60000` (1분) | 3600000 |
| `jwt.refresh-token-validity` | `JWT_REFRESH_TOKEN_VALIDITY` | `300000` (5분) | 86400000 |

> 테스트 완료 후 두 환경변수를 반드시 제거해 기본값으로 원복할 것

### 테스트 계정 준비

| 계정 | userType | 용도 |
|---|---|---|
| dev 포털 본사 계정 | C | TC-1 ~ TC-5 |
| dev 포털 협력사 계정 | S | TC-6 |
| dev 포털 관리자 계정 | A | TC-7 (선택) |
| `use_yn = 'N'` 계정 | C | TC-8 (테스트 전후 DB 직접 변경) |

---

## TC-1. 정상 Refresh Rotation

**목적**: Access Token 만료 시 자동 갱신, 로그아웃 없이 계속 사용 가능한지 확인

**배포 묶음**: 묶음 1

**단계**:
1. 포털 웹에서 본사(C) 계정으로 로그인
2. DevTools → Application → Local Storage 열기
3. `refreshToken` 키 존재 확인, 값 복사해둠 (묶음 4 적용 시 `sessionId` 키는 없어야 함)
4. **1분 이상 대기** (Access Token 만료 시간)
5. 화면에서 데이터 조회 동작 수행 (목록 조회, 페이지 이동 등)
6. 로그아웃 화면으로 이동하지 않고 정상 응답 확인

**예상 결과**:
- [ ] 화면이 정상 동작 (로그아웃 없음)
- [ ] DevTools Network 탭에서 `/api/v1/auth/refresh` 호출 확인 (200 응답)
- [ ] 갱신 후 `refreshToken` 값이 바뀜 (rotation)
- [ ] 새 refresh 토큰의 만료 시간이 **최초 로그인 + 5분** (절대 만료 유지)
  - jwt.io에서 디코딩해 `exp` 확인
- [ ] dev 서버 로그: `Access Token 갱신 완료: userId=...`

---

## TC-2. 절대 만료 (로그인 후 5분 경과)

**목적**: Refresh Token 만료 후 강제 로그아웃되는지 확인

**배포 묶음**: 묶음 1

**단계**:
1. 포털 웹에서 본사(C) 계정으로 로그인
2. **5분 이상 대기** (Refresh Token 만료)
3. 화면에서 임의 동작 수행

**예상 결과**:
- [ ] 로그인 화면으로 이동
- [ ] dev 서버 로그: `유효하지 않은 Refresh Token`

---

## TC-3. 다른 기기 로그인 시 기존 세션 무효화

**목적**: 같은 계정으로 다른 브라우저 로그인 시 기존 세션 차단 확인 (#7)

**배포 묶음**: 묶음 1

**단계**:
1. **브라우저 A** (Chrome 일반): 본사(C) 계정으로 로그인
2. **브라우저 B** (Chrome 시크릿 or Edge): 동일 계정으로 로그인
3. 브라우저 A로 돌아와 **1분 이상 대기**
4. 브라우저 A에서 화면 동작 수행 (refresh 발생 유도)

**예상 결과**:
- [ ] 브라우저 A: "다른 기기에서 로그인" 안내 메시지 후 로그인 화면 이동
- [ ] 브라우저 B: 계속 정상 동작
- [ ] dev 서버 로그: `세션 무효화됨 (다른 기기에서 로그인): userId=..., sessionId=...`

---

## TC-4. 구 토큰 전환 (sessionId 없는 기존 토큰)

**목적**: 묶음 1 배포 직후 이미 로그인된 사용자(기존 refresh 토큰)가 정상 갱신되는지 확인

**배포 묶음**: 묶음 1 배포 직후 24시간 이내

**단계**:
1. 묶음 1 배포 **전**에 로그인한 상태로 유지
2. 묶음 1 배포 **후** 1분 대기
3. 화면 동작 수행

**예상 결과**:
- [ ] 로그아웃 없이 정상 갱신
- [ ] dev 서버 로그: `구 형식의 refresh 토큰 수신 (sessionId 없음, 전환 기간): userId=...`

> 이 로그가 배포 24시간 후에도 나타나면 출처 확인 필요

---

## TC-5. SUPPLIER refresh 후 수수료 수정 가능 여부

**목적**: refresh 후 발급된 Access Token에 `suppId` 클레임이 포함되어 수수료 수정 API 접근 가능한지 확인 (#28 관련)

**배포 묶음**: 묶음 1

**단계**:
1. 포털 웹에서 협력사(S) 계정으로 로그인
2. **1분 이상 대기** (Access Token 만료 → refresh 발생)
3. 화면에서 라이더 수수료 수정 시도
4. (선택) DevTools에서 갱신된 Access Token을 jwt.io로 디코딩

**예상 결과**:
- [ ] 수수료 수정 성공 (403 Forbidden 없음)
- [ ] 디코딩한 Access Token에 `suppId` 클레임 존재 확인
- [ ] dev 서버 로그: `Access Token 갱신 완료: userId=...`

---

## TC-6. 비활성 계정 Refresh 차단

**목적**: `use_yn = 'N'` 계정이 refresh 시 차단되는지 확인

**배포 묶음**: 묶음 1

**사전 작업**: dev DB에서 테스트 계정 `use_yn = 'N'` 으로 변경

**단계**:
1. `use_yn = 'N'` 계정으로 기존에 발급된 Refresh Token으로 직접 API 호출

   ```
   POST /api/v1/auth/refresh
   Body: { "refreshToken": "<기존_refresh_token>" }
   ```

2. (또는 웹에서 로그인 후 `use_yn = 'N'` 으로 변경 → 1분 대기 → 화면 동작)

**예상 결과**:
- [ ] 401 응답
- [ ] 로그인 화면 이동
- [ ] dev 서버 로그: `비활성화된 계정으로 refresh 시도: userId=...`

**사후 작업**: 테스트 계정 `use_yn = 'Y'` 복원

---

## TC-7. 비허용 userType(R) Refresh 차단

**목적**: 모바일 라이더 토큰으로 포털 refresh 엔드포인트 접근 차단 확인

**배포 묶음**: 묶음 1

**단계**:
1. 모바일 라이더 계정으로 발급된 Refresh Token 확보
2. Postman으로 직접 호출:

   ```
   POST /api/v1/auth/refresh
   Header: X-API-Secret: <secret>
   Body: { "refreshToken": "<라이더_refresh_token>" }
   ```

**예상 결과**:
- [ ] 401 응답
- [ ] dev 서버 로그: `포털 미허용 userType으로 refresh 시도: userId=..., userType=R`

---

## TC-8. ACCESS 토큰으로 Refresh 시도 차단

**목적**: Access Token을 refresh 엔드포인트에 넘기면 차단되는지 확인 (#6 관련)

**배포 묶음**: 묶음 2

**단계**:
1. 포털 로그인 후 Access Token 복사 (DevTools → Network → 임의 API 요청의 Authorization 헤더에서)
2. Postman으로 직접 호출:

   ```
   POST /api/v1/auth/refresh
   Header: X-API-Secret: <secret>
   Body: { "refreshToken": "<access_token_값>" }
   ```

**예상 결과**:
- [ ] 401 응답
- [ ] dev 서버 로그: `Refresh Token이 아닌 토큰으로 갱신 시도`

---

## TC-9. 로그인 실패 메시지 통일

**목적**: 포털·모바일 앱 로그인 실패 시 메시지가 통일됐는지 확인 (#27)

**배포 묶음**: 묶음 3

**단계 (포털 웹)**:
1. 포털에서 잘못된 비밀번호로 로그인 시도

**단계 (모바일 앱)**:
1. 앱에서 잘못된 비밀번호로 로그인 시도

**예상 결과**:
- [ ] 포털: "아이디 또는 비밀번호를 확인해주세요"
- [ ] 앱: 동일한 안내 UI 표시 (`useLogin.ts:122`의 `includes('비밀번호')` 분기 확인)

---

## TC-10. 로그 민감정보 제거 확인

**목적**: OTP 코드, API Secret, 비밀번호 재설정 토큰이 로그에 남지 않는지 확인 (#12 #13 #14 #21)

**배포 묶음**: 묶음 3

| 항목 | 재현 방법 | 확인할 로그 |
|---|---|---|
| #12 관리자 로그인 | MobAdminLoginController 로그인 | `userId`만 출력, Request 객체 전체 없음 |
| #13/#14 OTP 코드 | 라이더 앱 회원가입 / OTP 인증 | 6자리 OTP 코드 없음 |
| #14 reset token | 포털 비밀번호 재설정 | token 앞 8자 마스킹 (`tok****`) |
| #21 API Secret | 잘못된 Secret으로 API 호출 | `(length=N)`으로 표시, 앞 10자 노출 없음 |

**확인 방법**: dev 서버 CloudWatch Logs (또는 로컬 콘솔)에서 해당 액션 수행 후 로그 검색

---

## 배포 직후 모니터링 체크리스트

묶음 1 배포 후 **30분~1시간** CloudWatch Logs 확인:

| 로그 패턴 | 기대 | 이상 징후 |
|---|---|---|
| `API Secret 헤더 누락: POST /api/v1/auth/refresh` | 0건 | 발생 시 화이트리스트 미적용 |
| `Access Token 갱신 완료: userId=` | 건수 증가 | 0건 지속 시 refresh 미동작 |
| `구 형식의 refresh 토큰 수신` | 배포 직후 다수, 24h 후 0건 | 24h 후에도 지속 시 출처 확인 |
| `세션 검증 중 오류` | 0건 | 발생 시 Redis 상태 확인 |
| `포털 미허용 userType으로 refresh 시도` | 소수 | 급증 시 원인 확인 |
| `비활성화된 계정으로 refresh 시도` | 소수 | 급증 시 원인 확인 |

**롤백 기준**: 포털 사용자 로그아웃 문의 증가 또는 `세션 검증 중 오류` 지속 발생 → 묶음 1 전체 롤백

---

## 테스트 완료 후 정리

- [ ] dev ECS 임시 환경변수(`JWT_ACCESS_TOKEN_VALIDITY`, `JWT_REFRESH_TOKEN_VALIDITY`) 제거
- [ ] TC-6에서 변경한 `use_yn = 'N'` 계정 복원
