
#troubleshooting #relay-server #puppeteer #baemin #crawling #ecs #cpu #memory #logging #queue #keep-alive #bot-detection

---

## [이슈 1] relay-server 라이더 상세 조회 중 CPU/MEMORY 급증으로 ECS 태스크 중지 및 봇차단 재발

---

### 1. 문제 상황

**발생 일자**: 2026-04-22

**발생 시간**: 21:09 ~ 21:46 KST

**대상 서비스**: `vroong-relay-service`

**관련 로그 그룹**: relay-server ECS task log

21:09경 라이더 상세 정보 조회가 대량으로 수행되던 중 relay-server 태스크의 CPU/MEMORY 사용량이 급증했고, 이후 ECS 태스크가 중지되었다.
태스크 재시작 이후 Puppeteer 세션은 메모리에만 존재하므로 기존 세션을 이어받지 못했고, 세션이 없는 상태에서 배치성 API 호출이 계속 유입되었다.
그 결과 keep-alive가 세션 없음 상태를 감지하고 자동 재인증을 시도했으며, 배민 로그인 페이지에서 봇 탐지 차단이 발생했다.

```text
21:09  [Fast 라이더 상세] 완료/조회 로그 반복
21:11  relay-server 태스크 재시작
21:12  Redis active_sessions에는 값이 있으나 browserManager 로컬 세션 없음
21:14  세션 없는 상태에서 라이더 상세/배달현황 API 호출 재개
21:21  keep-alive가 세션 없음 감지 후 자동 재인증 시작
21:22  Puppeteer 로그인 중 봇 탐지 차단
21:22~  bot-detection cooldown 진입
```

---

### 2. 원인 분석

#### 2.1 태스크 중지 원인

라이더 상세 조회와 배달현황 조회가 짧은 시간에 많이 들어오면서 relay-server 내부 요청 큐에 대기 작업이 계속 쌓였다.
기존 구조에서는 세션별 큐 길이 제한이 없어서, 요청이 처리 속도보다 빠르게 유입될 경우 pending Promise, Express response context, centerId/riderId 정보가 메모리에 계속 누적될 수 있었다.

```text
[API 요청 폭증]
        -> [browserManager.enqueueRequest 대기열 증가]
        -> [pending request / closure / response context 누적]
        -> [CPU/MEMORY 상승]
        -> [ECS health check 실패 또는 리소스 부족]
        -> [태스크 중지]
```

#### 2.2 태스크 재시작 이후 세션 문제

Puppeteer 세션은 relay-server 프로세스 메모리와 browser context에 의존한다.
따라서 ECS 태스크가 재시작되면 Redis에 남아 있는 sessionId가 있더라도 실제 브라우저 세션은 사라진 상태가 된다.
이 상태에서 배치 API가 계속 호출되면 relay-server는 유령 세션 또는 세션 없음 상태로 동작하게 된다.

#### 2.3 봇차단 재발 원인

세션이 없는 상태에서 자동 재인증이 반복되거나, 배치 API 호출과 재인증 흐름이 겹치면 배민 보안 정책상 비정상 접근으로 판단될 가능성이 커진다.
특히 운영 IP에서 Puppeteer 로그인 시도가 실패하면 봇 탐지 쿨다운에 들어가며, 그 동안 자동 재인증을 반복하면 상태가 더 악화될 수 있다.

#### 2.4 로그 과다 문제

기존 relay-server는 센터/라이더 단위 API 호출마다 시작, 성공, 재시도, 실패 로그를 모두 출력했다.
대량 배치에서는 정상 로그만으로도 CloudWatch 로그가 과도하게 증가했고, 장애 시점의 핵심 실패 원인을 찾기 어려웠다.

---

### 3. 해결 방안

#### 3.1 browserManager 세션별 큐 제한 추가

`relay-server/services/browser/core.js`에 세션별 pending request 제한과 큐 대기 timeout을 추가했다.

```javascript
const MAX_SESSION_PENDING_REQUESTS = Number(process.env.MAX_SESSION_PENDING_REQUESTS || 20);
const REQUEST_QUEUE_MAX_WAIT_MS = Number(process.env.REQUEST_QUEUE_MAX_WAIT_MS || 120000);
```

`enqueueRequest()`에서 pending 요청이 임계치를 넘으면 새 요청을 받지 않고 즉시 `RELAY_QUEUE_FULL` 오류를 반환하도록 변경했다.

```javascript
const pendingCount = getRequestCount(sessionPendingRequestCounts, sessionId);
const activeCount = getRequestCount(sessionActiveRequestCounts, sessionId);

if (pendingCount >= MAX_SESSION_PENDING_REQUESTS) {
    const message =
        `RELAY_QUEUE_FULL: sessionId=${sessionId.substring(0, 8)}, active=${activeCount}, pending=${pendingCount}`;
    console.warn(`[BrowserManager] ${message}`);
    throw new Error(message);
}
```

큐에 너무 오래 대기한 요청도 실행하지 않고 `RELAY_QUEUE_TIMEOUT`으로 중단하도록 변경했다.

```javascript
const waitedMs = Date.now() - queuedAt;

if (waitedMs > REQUEST_QUEUE_MAX_WAIT_MS) {
    decrementRequestCount(sessionPendingRequestCounts, sessionId);
    throw new Error(
        `RELAY_QUEUE_TIMEOUT: sessionId=${sessionId.substring(0, 8)}, waitedMs=${waitedMs}`
    );
}
```

#### 3.2 relay busy 응답 처리 추가

`server.js`의 Fast API 라우터에서 `RELAY_QUEUE_FULL`, `RELAY_QUEUE_TIMEOUT` 발생 시 500이 아니라 503 `RELAY_BUSY`로 응답하도록 정리했다.

```javascript
function isRelayBusyError(error) {
    const message = error?.message || '';
    return message.includes('RELAY_QUEUE_FULL') || message.includes('RELAY_QUEUE_TIMEOUT');
}

function handleFastApiError(res, label, error, context = {}) {
    const message = error?.message || 'unknown error';

    if (isRelayBusyError(error)) {
        console.warn(`[${label} 실패] error=${message}`);
        return res.status(503).json({
            success: false,
            error: 'RELAY_BUSY',
            message
        });
    }

    console.error(`[${label} 실패] error=${message}`);
    return res.status(500).json({ success: false, error: message });
}
```

#### 3.3 Fast API timeout/retry 조정

`relay-server/services/baemin/fast-api.js`에서 Fast API timeout과 retry 값을 환경변수로 제어할 수 있게 정리했다.

```javascript
const MAX_RETRIES = Number(process.env.FAST_API_MAX_RETRIES || 2);
const RETRY_DELAY_MS = Number(process.env.FAST_API_RETRY_DELAY_MS || 1000);
const FAST_API_TIMEOUT_MS = Number(process.env.FAST_API_TIMEOUT_MS || 15000);
const FAST_API_FILE_TIMEOUT_MS = Number(process.env.FAST_API_FILE_TIMEOUT_MS || 45000);
```

파일 다운로드 API에서는 timeout 키가 실제로 적용되도록 수정했다.

```javascript
lastResult = await directApiCallForFile(page, endpoint, {
    centerId,
    timeout: FAST_API_FILE_TIMEOUT_MS
});
```

#### 3.4 개별 센터/라이더 실패는 배치 전체 중단 없이 계속 진행

센터 또는 라이더 단위의 400/403/404/500/502/503/504 실패는 즉시 전체 배치를 중단하지 않고, 해당 대상만 빈값 또는 null로 반환하도록 유지했다.
단, 세션 만료/401/큐 과부하/큐 대기 timeout은 전체 흐름에 영향을 주는 오류이므로 상위로 전파한다.

| 오류 유형 | 처리 방식 |
|----------|----------|
| 400/404 데이터 없음 | 빈 배열 또는 null 반환 |
| 403 일시 차단 | 빈 배열 또는 temporary block 처리 |
| 500/502/503/504 개별 실패 | 실패 로그 후 빈값 반환 |
| 401/SESSION_EXPIRED | 상위로 throw |
| RELAY_QUEUE_FULL | 503 RELAY_BUSY |
| RELAY_QUEUE_TIMEOUT | 503 RELAY_BUSY |

#### 3.5 keep-alive 로그 축소

keep-alive는 1분마다 실행되므로 정상 주기 로그를 모두 제거하거나 `debugLog()`로 전환했다.
운영에서는 `DEBUG_RELAY=false` 기준으로 실패성 로그만 남긴다.

유지하는 로그:

```text
[Keep-Alive] 세션 만료
[Keep-Alive] 복구 시도 실패
[Keep-Alive] 크롤링 장애 임계치 도달
[Keep-Alive] 자동 재인증 실패
[Keep-Alive] browserManager 세션 정리 실패
[Keep-Alive] 오류
```

정상 실행 로그, 세션 존재 로그, skip 로그, 스케줄러 시작/중지 로그는 `debugLog()`로 변경했다.

#### 3.6 keep-alive 협력사 목록 조회 silent 처리

keep-alive에서 세션 유효성 확인을 위해 협력사 목록을 조회할 때는 로그가 나오지 않도록 `silent` 옵션을 추가했다.

```javascript
async function getCenterListFast(session, options = {}) {
    const silent = options.silent === true;
    ...
}
```

keep-alive 호출부는 아래처럼 변경했다.

```javascript
const result = await getCenterListForKeepAlive(session, { silent: true });
```

직접 `/relay/center/list`를 호출할 때만 시작/종료/실패 로그가 남는다.

#### 3.7 API별 시작/종료/실패 로그 표준화

relay-server의 주요 API 라우터 로그를 다음 규칙으로 통일했다.

- 시작 로그: 호출 시간, centerId/riderId/page/fromDate/toDate 등 식별값
- 성공 로그: 종료 시간, 조회 데이터 수, elapsedMs
- 실패 로그: centerId/riderId 등 실패 대상, 오류 메시지, elapsedMs
- 제거 로그: sessionId 출력, 재시도별 성공/실패 로그, 센터/라이더별 정상 진행 로그

적용 대상:

```text
/relay/center/list
/relay/delivery/status
/relay/delivery/daily
/relay/delivery/rider
/relay/rider/info
/relay/rider/info/parallel
/relay/rider/detail/:riderId
/relay/partner/detail
/relay/partner/delivery-fee
/relay/fast/delivery/status
/relay/fast/delivery/daily
/relay/fast/delivery/rider
/relay/fast/rider/info
/relay/fast/rider/info/batch
/relay/fast/partner/detail
/relay/fast/rider/detail/:riderId
/relay/fast/partner/delivery-fee
```

예시:

```javascript
console.log(
    `[Fast 라이더 상세] 시작: startedAt=${startedAtText}, centerId=${centerId || 'none'}, riderId=${riderId || 'none'}`
);

console.log(
    `[Fast 라이더 상세] 성공: endedAt=${endedAtText}, centerId=${centerId}, riderId=${riderId}, count=${count}, elapsedMs=${endedAt - startedAt}`
);

console.error(
    `[Fast 라이더 상세] 실패: centerId=${centerId || 'none'}, riderId=${riderId || 'none'}, error=${message}, elapsedMs=${endedAt - startedAt}`
);
```

#### 3.8 getRiderInfoBatchFast 결과 구조 정리

`getRiderInfoBatchFast()`는 센터별 실패를 식별할 수 있도록 결과를 아래 구조로 통일했다.

```javascript
results[centerId] = {
    success: true,
    data
};
```

실패 시:

```javascript
results[centerId] = {
    success: false,
    data: [],
    error: error.message
};
```

서비스 내부의 전체 완료 로그는 제거하고, 전체 시작/종료/totalRiders/failedCount는 `/relay/fast/rider/info/batch` 라우터에서만 남긴다.

---

### 4. 관련 파일

| 파일 | 수정 내용 |
|------|----------|
| `relay-server/services/browser/core.js` | 세션별 pending queue 제한, queue timeout, active/pending count 관리 |
| `relay-server/services/baemin/fast-api.js` | Fast API timeout/retry 환경변수화, 내부 로그 축소, 센터별 실패 결과 구조 정리 |
| `relay-server/services/baemin/center.js` | keep-alive용 silent 옵션, 협력사 목록 내부 로그 축소 |
| `relay-server/services/baemin/delivery.js` | 배달현황/일별/라이더별 배달내역 재시도 상세 로그 제거, 최종 실패 로그만 유지 |
| `relay-server/services/baemin/rider.js` | 라이더 정보/상세/병렬 조회 로그 축소, 실패 대상 centerId/riderId 로그 유지 |
| `relay-server/services/baemin/partner.js` | 협력사 상세/배달비 로그 축소, 실패 대상 centerId 로그 유지 |
| `relay-server/server.js` | API별 시작/종료/실패 로그 표준화, keep-alive 로그 축소, relay busy 응답 처리 |
| `api/src/main/java/com/vroong/api/infrastructure/crawling/webclient/RelayServerClient.java` | relay busy/session unavailable 응답 처리 대상 |
| `api/src/main/java/com/vroong/api/infrastructure/crawling/webclient/RiderWebClient.java` | 라이더 상세 조회 실패 재시도/스킵 처리 대상 |
| `api/src/main/java/com/vroong/api/domain/crawling/baemin/rider/usecase/RiderDetailInfoUseCase.java` | 라이더 상세 조회 실패 라이더 재시도 및 병렬 처리 제어 대상 |

---

### 5. 운영 환경변수 권장값

```env
MAX_SESSION_PENDING_REQUESTS=20
REQUEST_QUEUE_MAX_WAIT_MS=120000
FAST_API_MAX_RETRIES=2
FAST_API_RETRY_DELAY_MS=1000
FAST_API_TIMEOUT_MS=15000
FAST_API_FILE_TIMEOUT_MS=45000
DEBUG_RELAY=false
```

운영에서 장애 원인 추적이 필요할 때만 일시적으로 `DEBUG_RELAY=true`를 사용한다.
상시 운영에서는 정상 진행 로그를 줄이기 위해 `false`를 유지한다.

---

### 6. 기대 효과

| 항목 | 개선 효과 |
|------|----------|
| 큐 제한 | 요청 폭증 시 메모리 무한 증가 방지 |
| queue timeout | 오래 대기한 요청이 뒤늦게 실행되는 문제 방지 |
| 503 RELAY_BUSY | relay 과부하를 API 서버가 구분 가능 |
| 실패 대상 로그 | 어떤 centerId/riderId에서 실패했는지 추적 가능 |
| 로그 축소 | CloudWatch 로그 비용 및 장애 분석 난이도 감소 |
| keep-alive silent | 정상 주기 로그 제거, 실제 장애 로그만 확인 가능 |
| 세션 없음 방어 | 태스크 재시작 직후 비정상 크롤링 요청 완화 |

---

### 7. 후속 조치

- [x] 세션별 relay queue 제한 추가
- [x] queue wait timeout 추가
- [x] Fast API timeout/retry 환경변수화
- [x] keep-alive 정상 로그 제거 및 실패 로그 중심 변경
- [x] keep-alive 협력사 목록 조회 silent 처리
- [x] 주요 relay API 시작/종료/실패 로그 표준화
- [x] 센터/라이더 단위 실패 로그에 centerId/riderId 포함
- [ ] 운영 ECS 환경변수 반영 확인
- [ ] 배치 시간대 CPU/MEMORY CloudWatch 지표 재확인
- [ ] RELAY_BUSY 발생 빈도 모니터링
- [ ] 봇차단 재발 여부 모니터링

---

### 8. 결론

이번 장애의 직접 원인은 라이더 상세/배달현황 대량 조회 중 relay-server의 CPU/MEMORY가 급증하여 ECS 태스크가 중지된 것이다.
태스크 재시작 후 Puppeteer 세션이 사라진 상태에서 크롤링 API와 자동 재인증이 이어지면서 봇차단이 재발했다.

수정 방향은 크게 세 가지다.

1. relay-server가 처리 가능한 요청만 받도록 queue 제한을 둔다.
2. 세션 없음/과부하/개별 실패를 구분해서 응답하고 로그를 남긴다.
3. 정상 진행 로그를 제거하고 실패 대상 중심 로그만 남긴다.

이 변경으로 relay-server는 과부하 시 즉시 `RELAY_BUSY`를 반환하고, 운영 로그에서는 어떤 협력사 또는 라이더에서 실패했는지 중심으로 확인할 수 있다.
