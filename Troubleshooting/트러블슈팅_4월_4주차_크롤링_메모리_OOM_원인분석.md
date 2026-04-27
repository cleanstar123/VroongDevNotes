#troubleshooting #relay-server #puppeteer #baemin #crawling #ecs #memory #oom #chrome #gc #batch

---

## [이슈] relay-server 크롤링 메모리 OOM — 원인 분석 및 조치 내역

---

### 1. 문제 상황

**발생 일자**: 2026-04-26 (매일 반복)

**대상 서비스**: `vroong-relay-service` (ECS Fargate)

**증상 요약**

- relay-server ECS 태스크 메모리가 시간이 지날수록 지속적으로 우상향
- 매시 :10분 전후 메모리 스파이크 발생
- 매일 메모리 한계 초과로 ECS 태스크 강제 종료(OOM Kill) → 재시작

**실측 피해 (2026-04-26 rider_detail_info_if 기준)**

| if_date_seq | 시간대 | duration_seconds | 비고 |
|---|---|---|---|
| 1~10 | 01시~13시 | 228~250초 (정상) | 약 4분 소요 |
| 11 | 14:11~15:19 | **4,032초 (67분)** | OOM Kill 후 재시작 구간 |
| 15:xx | - | **누락** | 서버 재시작 중 배치 실패 |
| 12~16 | 16시~20시 | 243~260초 (정상) | 재시작 후 정상화 |

---

### 2. 원인 분석

#### 2-1. relay-server가 Chrome을 사용하는 구조

relay-server는 배민 공식 API가 없어 **Puppeteer(Chrome 자동화)** 로 크롤링합니다.
Node.js에서 직접 HTTP를 보내는 게 아니라, Chrome 브라우저 안에서 `fetch()`를 실행하는 방식입니다.

```
api-server → relay-server → page.evaluate(fetch(...)) → Chrome V8 → 배민 API
```

`page.evaluate()` 호출 시 JSON 응답 객체가 Chrome V8 힙에 생성된 뒤 Node.js로 전달됩니다.

#### 2-2. Chrome GC가 실행되지 않는 이유

Chrome의 GC(Garbage Collection)는 **한가할 때(idle time)** 실행됩니다.
배치 크롤링이 연속으로 실행되는 동안 Chrome은 계속 바쁜 상태이므로 GC가 실행되지 못하고 V8 힙이 누적됩니다.

```
작업이 계속 들어옴 → Chrome 바쁨 → GC idle time 없음 → V8 힙 미해제 → 메모리 우상향
```

#### 2-3. 매시 :10분 스파이크의 정확한 원인

`riderInfoTrigger` (`0 10 0-3,7-23 * * ?`) 가 **매시 :10분**에 전체 센터 라이더 정보를 크롤링합니다.

```java
// RiderUseCase.java
List<Center> centers = centerList.getCenters();  // 전체 센터 (약 100개)

for (Center center : centers) {
    riderWebClient.fetchRiderInfo(session, centerId);  // 센터마다 relay-server 호출
}
```

센터 100개 × `page.evaluate(fetch(...))` = 100회 연속 V8 힙 객체 생성 → GC 기회 없음 → 스파이크

#### 2-4. 타이머 기반 정리가 배치 중 동작하지 않는 이유

기존에 20분마다 CDP `HeapProfiler.collectGarbage`를 실행하는 코드가 있었으나,
배치 중에는 세션이 항상 `hasBusySession = true` 상태이므로 **정리가 매번 건너뜀**.

```js
// 타이머 기반 정리 — 배치 중에는 실질적으로 실행되지 않음
if (hasBusySession(sessionId)) {
    continue;  // 항상 건너뜀
}
await clearChromeMemory(session.page);
```

---

### 3. 추가 확인: dev vs prod 차이

동일 조건 확인 결과 (2026-04-26 기준):

| 항목 | dev | prod | 비고 |
|---|---|---|---|
| 협력사(센터) 수 | 63개 | 63개 | 동일 |
| partner_details_if | 63건 | 63건 | 동일 |
| rider_delivery_histories_if | 1,372건 | 2,866건 | 2x |
| **partner_delivery_fees_if** | **14,977건** | **32,174건** | **2.15x** |

**실측 메모리 패턴 비교 (09:00 배치 기준)**

| 시각 | dev | prod |
|---|---|---|
| 08:55 | 39.0% | 63.4% |
| 09:00 | 41.2% | 68.75% |
| 09:05 | 40.0% | 65.5% |
| 09:10 | 39.2% | 69.2% |
| 09:15 | 39.0% | 69.5% |
| 09:25 | — | 68.1% |

dev는 배치 후 완전 회복. prod는 스파이크 후 기저선으로 돌아오지 않고 계속 우상향.

---

**원인 A — prod 전용 페이지 재시작 로직이 역효과**

dev 브랜치에는 `restartSessionPage`, `recycleSessionPageIfNeeded` 코드가 없음. 페이지는 로그인 이후 세션 종료 시까지 한 번만 생성됨.

prod 브랜치에는 50요청마다 `recycleSessionPageIfNeeded`가 실행:

```
1. 기존 페이지 닫기  → V8 힙 해제 ✅
2. 새 페이지 생성    → blank
3. deliverycenter.baemin.com/center/change 로 navigate
   → 배민 SPA (React 앱, JS 번들, CSS) 전체 로드  ⚠️
```

`--aggressive-cache-discard` 플래그로 캐시가 비활성화되어 있어, 매 페이지 재시작마다 배민 SPA 자산을 새로 다운로드함. 배치 사이 "유휴 상태"에도 항상 갓 로드된 SPA가 Chrome 렌더러 메모리를 점유 → 메모리가 회복되지 않음.

dev는 페이지 재시작 없음 → 배치 후 Chrome이 유휴 GC를 실행 → 메모리 회복.

```
dev  → 페이지 재시작 없음 → 배치 후 유휴 GC → 메모리 회복
prod → 50요청마다 SPA 재로드 → 렌더러 메모리 계속 점유 → 메모리 우상향
```

**원인 B — Excel 파일 base64 변환의 O(n²) 특성 (9시, 10시 한정)**

`directApiCallForFile` 함수 내부에서 Excel ArrayBuffer를 base64로 변환할 때 문자열 연결 방식 사용:

```js
// fast-api.js — directApiCallForFile 내부 (Chrome V8 안에서 실행)
let binary = '';
for (let i = 0; i < bytes.length; i++) {
    binary += String.fromCharCode(bytes[i]);  // ← O(n²) 문자열 연결
}
const base64 = btoa(binary);
```

prod Excel 파일이 dev보다 2.15배 크므로 V8 힙 사용량은 약 4배(2² = 4) 증가.
63개 센터 × 2회(9시, 10시) = 126회 발생 → 9시, 10시 스파이크에 기여.

**원인 C — prod 코드에만 존재했던 메모리 누수**

- `waitForSessionReady` 타이머 누수 — 매 API 호출마다 15초 타이머 미해제 → 누적 (수정 완료)
- `sessionNavTimeoutCount` Map — prod에만 추가된 코드로 세션 타임아웃 카운트 누적

---

### 4. 조치 내역

#### 조치 1 — `waitForSessionReady` 타이머 누수 수정 (배포 완료)

매 API 호출마다 15초 타이머가 생성되고 해제되지 않는 버그.

```js
// 수정 전: clearTimeout 누락
const timeoutPromise = new Promise((_, reject) => {
    setTimeout(() => reject(...), timeout);  // ← 해제 안 됨
});

// 수정 후: clearTimeout 명시
let timeoutId;
const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => reject(...), timeout);
});
try {
    await Promise.race([promise, timeoutPromise]);
    clearTimeout(timeoutId);  // ← 성공 시 해제
} catch (error) {
    clearTimeout(timeoutId);  // ← 실패 시도 해제
    throw error;
}
```

#### 조치 2 — Chrome 메모리 절약 플래그 추가 (배포 완료)

```js
'--disable-extensions', '--disable-plugins-discovery',
'--aggressive-cache-discard', '--disable-background-networking',
'--disable-default-apps', '--disable-sync',
'--metrics-recording-only', '--mute-audio',
```

#### 조치 3 — 페이지 주기적 재생성 비활성화 (SESSION_PAGE_RECYCLE_REQUESTS=0)

기존 50요청마다 페이지를 재시작하는 로직이 배민 SPA를 반복 로드하여 오히려 메모리를 증가시키는 역효과 확인.
`CHROME_GC_INTERVAL_REQUESTS=10`(조치 4)이 V8 힙을 직접 처리하므로 페이지 재시작 불필요 → 비활성화.

#### 조치 4 — 요청 사이 간격 CDP GC 실행

배치 중 타이머 방식이 동작하지 않는 문제를 해결하기 위해, **`enqueueRequest` finally 블록**에서 10요청마다 직접 GC 실행.

```js
// core.js — enqueueRequest finally 블록
const completedCount = incrementCompletedRequestCount(sessionId);

if (CHROME_GC_INTERVAL_REQUESTS > 0 && completedCount % CHROME_GC_INTERVAL_REQUESTS === 0) {
    const currentPage = sessionPages.get(sessionId)?.page;
    await collectChromeGarbage(currentPage);  // HeapProfiler.collectGarbage
}
```

- 배치 시간 추가 오버헤드: 약 1.5초 (100센터 기준, 0.6%)
- 페이지 탐색 없음, 쿠키/세션 유지, 봇 탐지 영향 없음

#### 조치 5 — Chrome 메모리 정리 타이머 20분 → 3분

유휴 상태(배치 사이)에서 더 빠르게 기저 메모리 회복.

```js
// 수정 전
const CHROME_MEMORY_CLEANUP_INTERVAL = 20 * 60 * 1000;  // 20분

// 수정 후
const CHROME_MEMORY_CLEANUP_INTERVAL = 3 * 60 * 1000;   // 3분
```

---

### 5. 예상 효과

| 시나리오 | 조치 전 | 조치 후 (예상) |
|---|---|---|
| 매시 :10분 스파이크 | V8 힙 급등 | 10요청마다 GC로 억제 |
| 14시 OOM Kill | 매일 발생 | 스파이크 감소로 임계점 도달 방지 |
| 배치 후 메모리 회복 | 회복 안 됨 (SPA 재로드로 점유) | 페이지 재시작 없음 → 유휴 GC로 회복 |
| 유휴 시 메모리 회복 속도 | 20분 후 | 3분 후 |
| 배치 완료 시간 | 변화 없음 | +1.5초 (0.6%) |

---

### 6. 구조적 한계 (고객사 공유 필요)

배민이 공식 파트너 API를 제공하지 않아 **브라우저 자동화(Puppeteer)** 방식을 사용하는 구조적 한계가 있습니다.

- AWS ECS IP를 배민 서버가 봇으로 인식할 수 있음
- 배민 정책 변경 시 언제든 세션 차단 가능 → 크롤링 전면 중단 위험
- 메모리 문제는 완화되지만 Chrome 기반 크롤링인 이상 완전 해결은 어려움

**제안 사항**

1. 배민 파트너스 공식 API 사용 가능 여부 검토
2. 크롤링 실패 시 알림 → 수동 보완 프로세스 수립
3. 세션 차단 발생 시 대응 매뉴얼 준비

---

### 7. 관련 파일

| 파일 | 변경 내용 |
|---|---|
| `relay-server/services/browser/core.js` | GC 관련 모든 수정 |
| `relay-server/services/baemin/fast-api.js` | 세션 만료 플래그 관리 |
| `relay-server/server.js` | sessionNavTimeoutCount 정리, keep-alive 로직 |
| `.aws/update-relay-env.js` | 환경변수 자동 등록 스크립트 (SESSION_PAGE_RECYCLE_REQUESTS=0 추가) |
