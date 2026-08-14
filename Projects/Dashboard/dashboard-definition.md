# 대시보드 기능 정의서

> **작성일:** 2026-08-14  
> **대상 파일 경로:** `src/features/dashboard/`  
> **작성 목적:** 개발정의서 작성 사전 정리 — 기능, 데이터 구조, 컴포넌트, 비즈니스 로직 전반 문서화

---

## 목차

1. [개요](#1-개요)
2. [디렉토리 구조](#2-디렉토리-구조)
3. [화면 구성](#3-화면-구성)
4. [API 명세](#4-api-명세)
5. [데이터 타입](#5-데이터-타입)
6. [컴포넌트 상세](#6-컴포넌트-상세)
7. [비즈니스 로직](#7-비즈니스-로직)
8. [상태 관리 및 데이터 흐름](#8-상태-관리-및-데이터-흐름)
9. [설정 파일](#9-설정-파일)
10. [유틸리티 함수](#10-유틸리티-함수)
11. [CSS 구조](#11-css-구조)
12. [권한 및 역할별 차이](#12-권한-및-역할별-차이)

---

## 1. 개요

대시보드는 협력사별 배달 수행현황을 조회하는 핵심 화면입니다.

### 주요 기능

| 기능 | 설명 |
|---|---|
| 실시간/일별 현황 | 특정 날짜 기준 협력사별 완료건수 · 수락률 · 슬롯 현황 조회 |
| 주별 현황 | 수요일~화요일 기준 주간 협력사별 수행률/건수 조회 |
| 협력사 필터 | 본사 유저: 협력사 선택, 완료건수 이하 제외, 수락률 이하 제외 |
| 정렬 | 실시간 모드에서 완료건수 · 수락률 · 거절/취소건수 기준 3단계 정렬 |
| 요약 카드 | 활성 협력사 수 · 총 완료건수 · 수락률 · 슬롯달성률 |
| 멀티탭 | 각 기능 페이지를 탭 형태로 독립 관리, 세션 간 탭 상태 유지 |
| 공지 팝업 | 읽지 않은 팝업 공지사항 모달 표시 (오늘 하루 숨기기 지원) |
| 점검 모드 | 점검 중일 때 협력사에게 점검 화면 표시, 대시보드 외 탭 비활성화 |

### 사용자 유형별 접근

| USER_TYPE | 대시보드 접근 | 협력사 선택 필터 | 잔액 표시 |
|---|---|---|---|
| HEADQUARTERS | 전체 협력사 조회 가능 | 표시 (전체 또는 개별 선택) | 미표시 |
| SUPPLIER | 본인 협력사만 조회 | 미표시 | 표시 |
| RIDER | 해당 없음 | 해당 없음 | 미표시 |

---

## 2. 디렉토리 구조

```
src/features/dashboard/
├── api/
│   ├── dashboardApi.ts          # 대시보드 데이터 API
│   └── balanceApi.ts            # 잔액/선불금 API
│
├── components/
│   ├── layout/
│   │   ├── Header/
│   │   │   └── Header.tsx       # 상단 헤더 (유저 정보, 잔액, 로그아웃)
│   │   ├── Sidebar/
│   │   │   └── Sidebar.tsx      # 좌측 네비게이션 (역할별 메뉴)
│   │   └── Footer/
│   │       ├── Footer.tsx       # 하단 푸터 (개인정보처리방침 링크)
│   │       └── Footer.css
│   └── pages/
│       └── DashboardPage/
│           ├── DashboardPage.tsx             # 최상위 레이아웃 컨테이너
│           ├── DashboardTabBar.tsx           # 탭 바 (탭 전환 · 닫기)
│           ├── DashboardTabRouter.tsx        # 탭별 페이지 동적 라우팅 (26+ 페이지)
│           ├── DashboardMainContent.tsx      # 협력사별 수행현황 메인 뷰
│           ├── DashboardFilterSection.tsx    # 접을 수 있는 필터 패널
│           ├── DashboardSummaryCards.tsx     # 요약 KPI 카드 4개
│           ├── WeeklyPerformanceTable.tsx    # 주별 수행현황 테이블
│           └── RealtimePerformanceTable.tsx  # 실시간/일별 수행현황 테이블
│
├── config/
│   ├── filterKeys.config.ts     # 탭 닫을 때 삭제할 localStorage 키 매핑
│   ├── navigation.config.ts     # 사이드바 메뉴 트리 정의
│   └── pageLabels.config.ts     # 페이지 ID → 탭 라벨 매핑
│
├── types/
│   ├── dashboard.types.ts       # 주요 데이터 타입 정의
│   └── balance.types.ts         # 잔액 응답 타입
│
├── utils/
│   ├── dashboardUtils.ts        # 날짜 · 숫자 포맷 · 슬롯 유틸
│   └── noticeStorageUtils.ts    # 공지 오늘 하루 숨기기 로컬스토리지 관리
│
└── styles/
    ├── DashboardPage.css        # 레이아웃 · 테이블 · 필터 · 카드 스타일
    ├── Header.css               # 헤더 전용 스타일
    └── Sidebar.css              # 사이드바 전용 스타일
```

---

## 3. 화면 구성

### 레이아웃

```
┌─────────────────────────────────────────────────┐
│  Header (로고 | 탭바 | 잔액 | 유저정보)              │
├──────────┬──────────────────────────────────────┤
│          │  페이지 헤더 (제목 + 실시간/주별 토글)     │
│ Sidebar  ├──────────────────────────────────────┤
│ (메뉴)   │  요약 카드 × 4 (활성협력사·완료건수 등)   │
│          ├──────────────────────────────────────┤
│          │  필터 패널 (접기/펼치기)                 │
│          ├──────────────────────────────────────┤
│          │  협력사별 수행현황 헤더 + 건수/수행률 토글 │
│          ├──────────────────────────────────────┤
│          │  수행현황 테이블 (주별 or 실시간)         │
└──────────┴──────────────────────────────────────┘
│  Footer                                          │
└─────────────────────────────────────────────────┘
```

### 뷰 모드별 차이

| 항목 | 실시간/일별 | 주별 |
|---|---|---|
| 날짜 선택 | 단일 날짜 선택 (date input) | WeekSelector (수~화 자동 계산) |
| 협력사 필터 보임 여부 | 필터 패널 내 협력사 선택(HQ) | 동일 |
| 표시 모드 토글 | 없음 | 건수 / 수행률 전환 가능 |
| 테이블 컴포넌트 | `RealtimePerformanceTable` | `WeeklyPerformanceTable` |
| 정렬 기능 | 완료건수 · 수락률 · 거절건수 | 없음 |
| 완료건수 필터 기준 | 해당 날짜 단일 합계 | 기간 내 전체 합계 |

---

## 4. API 명세

### 4-1. dashboardApi — `/api/v1/dashboard`

#### `getSummary(params)`
- **Method:** GET
- **Path:** `/api/v1/dashboard/weekly`
- **Purpose:** 요약 카드용 데이터 (1분마다 자동 갱신)
- **Request Params:**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| userType | string? | 사용자 유형 |
| userId | string? | 사용자 ID |

- **Response:** `DashboardSummaryResponse`

---

#### `getWeekly(params)`
- **Method:** GET
- **Path:** `/api/v1/dashboard/weekly`
- **Purpose:** 협력사별 슬롯 · 건수 · 라이더 수행현황 (주별 / 실시간 공용)
- **Request Params:**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| fromDate | string | 시작일 (YYYYMMDD) |
| toDate | string | 종료일 (YYYYMMDD) |
| suppId | string? | 협력사 ID (미입력 시 전체) |

- **실시간 모드 호출:** `fromDate = toDate = 오늘`
- **주별 모드 호출:** `fromDate = 해당주 수요일`, `toDate = 다음주 화요일`
- **Response:** `DashboardWeeklyResponse`

---

### 4-2. balanceApi — `/api/v1/balance`

#### `getBalance()`
- **Method:** GET
- **Path:** `/api/v1/balance`
- **Purpose:** 협력사 잔액(선불금) 조회 — 헤더에 표시
- **Response:** `BalanceResponse`

---

## 5. 데이터 타입

### `DashboardSummaryResponse`

```typescript
interface DashboardSummaryResponse {
  deliveryDate: string;         // 기준 배달일 (YYYYMMDD)
  totFinishCnt: number;         // 전체 완료 건수
  totRejectCnt: number;         // 전체 거절 건수
  totAssignCancelCnt: number;   // 배차 취소 건수
  totDeliCancelCnt: number;     // 배달 취소 건수
  totRiderCnt: number;          // 전체 라이더 수
  totMorTimeCnt?: number;       // 아침점심피크 건수
  totAfterNonCnt?: number;      // 오후논피크 건수
  totNightCnt?: number;         // 저녁피크 건수
  totNightNonCnt?: number;      // 심야논피크 건수
  totMorTimeSlot?: number;      // 아침점심피크 슬롯 수
  totAfterNonSlot?: number;     // 오후논피크 슬롯 수
  totNightSlot?: number;        // 저녁피크 슬롯 수
  totNightNonSlot?: number;     // 심야논피크 슬롯 수
  totSlotSet?: number;          // 총 슬롯셋 수
  statusSummary: DashboardStatusSummaryItem[];
  success: boolean;
  message: string;
}
```

---

### `DashboardWeeklyResponse`

```typescript
interface DashboardWeeklyResponse {
  weekStartDate: string;                // 주 시작일 (YYYYMMDD)
  weekEndDate: string;                  // 주 종료일 (YYYYMMDD)
  totalFinishCnt: number;              // 기간 총 완료 건수
  activeSupplierCount: number;         // 활성 협력사 수
  periodSlotAchievementRate: number;   // 기간 슬롯 달성률 (%)
  periodAcceptanceRate: number;        // 기간 수락률 (%)
  periodMorTimeCnt: number;            // 아침점심피크 건수
  periodAfterNonCnt: number;           // 오후논피크 건수
  periodNightCnt: number;              // 저녁피크 건수
  periodNightNonCnt: number;           // 심야논피크 건수
  periodMorTimeRate: number;           // 아침점심피크 달성률
  periodAfterNonRate: number;          // 오후논피크 달성률
  periodNightRate: number;             // 저녁피크 달성률
  periodNightNonRate: number;          // 심야논피크 달성률
  dailyPerformances: DashboardWeeklyDailyPerformance[];
  supplierPerformances?: DashboardWeeklySupplierPerformance[];
  success: boolean;
  message: string;
}
```

---

### `DashboardWeeklySupplierPerformance`

```typescript
interface DashboardWeeklySupplierPerformance {
  suppId: string;
  suppName: string;
  slotSetCnt?: number;              // 슬롯셋 수
  totalCnt: number;                 // 총 배달 건수
  finishCnt: number;                // 완료 건수
  rejectCnt?: number;               // 거절 건수
  assignCancelCnt?: number;         // 배차 취소 건수
  deliCancelCnt?: number;           // 배달 취소 건수
  riderCount: number;               // 라이더 수
  acceptanceRate: number;           // 수락률 (0~100)

  // 슬롯별 달성률 — 키: YYYYMMDD, 값: 슬롯별 달성률 배열
  // 예) {"20241216": ["100%", "50%", "0%", "—"]}
  dailySlots: Record<string, string[]>;

  // 슬롯별 건수 — 키: YYYYMMDD, 값: "완료/목표" 형식 배열
  // 예) {"20241216": ["10/20", "5/10", "0/10", "—"]}
  dailySlotCounts: Record<string, string[]>;

  slotNames?: string[];             // 슬롯 이름 배열
  slotTimeRanges?: string[];        // 슬롯 시간대 배열
  dailySlotNames?: Record<string, string[]>;
  dailySlotTimeRanges?: Record<string, string[]>;
  standardAreas?: string[];         // 표준구역 목록
  dailyMissionRate?: Record<string, string>;  // 미션 모드 결과
}
```

---

### `Tab`

```typescript
interface Tab {
  id: string;       // 탭 고유 ID (타임스탬프 기반)
  label: string;    // 탭 표시 라벨
  page: string;     // 연결 페이지 ID (e.g. 'dashboard', 'rider_status')
}
```

---

### `BalanceResponse`

```typescript
interface BalanceResponse {
  balance: number;  // 잔액 (원 단위)
}
```

---

## 6. 컴포넌트 상세

### 6-1. `DashboardPage`

최상위 컨테이너. 전체 레이아웃(사이드바 + 헤더 + 탭바 + 콘텐츠 영역)을 관리.

**주요 상태**

| 상태 | 타입 | 설명 |
|---|---|---|
| `currentPage` | string | 현재 활성 페이지 ID |
| `tabs` | Tab[] | 열려 있는 탭 목록 |
| `activeTabId` | string | 현재 활성 탭 ID |
| `userInfo` | object | `{ userName, userEmail, userType }` |
| `popupNotices` | object[] | 표시할 팝업 공지 목록 |
| `maintenanceAcknowledged` | boolean | 점검 화면 확인 여부 |

**주요 동작**

| 함수 | 설명 |
|---|---|
| `handlePageChange(page)` | 새 페이지 탭 열기 (이미 열려 있으면 해당 탭으로 전환) |
| `handleTabClick(tabId)` | 기존 탭으로 전환 |
| `handleCloseTab(tabId)` | 탭 닫기 + 관련 localStorage 필터 키 정리 |
| `handleMaintenanceAcknowledge()` | 점검 화면 확인 처리 |

**초기화 순서**
1. localStorage에서 탭·페이지 상태 복원
2. `authApi.getUserInfo()` 호출 → 사용자 정보 세팅
3. `noticeApi.getUnreadPopupNotices()` 호출 → 팝업 공지 세팅

---

### 6-2. `DashboardMainContent`

협력사별 수행현황 뷰. 실시간/주별 모드 관리, 필터 · 정렬 처리.

**주요 상태**

| 상태 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `viewMode` | 'realtime' \| 'weekly' | 'realtime' | 뷰 모드 |
| `selectedDate` | string | 오늘 (06시 기준) | 실시간 기준 날짜 |
| `fromDate` / `toDate` | string | 이번 주 수~화 | 주별 조회 기간 |
| `weeklyData` | DashboardWeeklyResponse | null | 주별 API 결과 |
| `topAreaData` | DashboardWeeklyResponse | null | 상단 카드용 단일일 API 결과 |
| `selectedSupplierId` | string | '' (HQ) / 본인 ID (협력사) | 선택된 협력사 |
| `slotCountFilter` | string | '' | 완료건수 이상 필터 |
| `acceptRateFilter` | string | '' | 수락률 이상 필터 |
| `displayMode` | 'count' \| 'rate' | 'rate' | 주별 테이블 표시 단위 |
| `sortColumn` | SortColumn \| null | null | 정렬 기준 컬럼 |
| `sortDir` | 'asc' \| 'desc' | 'asc' | 정렬 방향 |

**API 호출 패턴**

```
초기 마운트 (fromDate/toDate 세팅 후):
  ① getWeekly({ fromDate: today, toDate: today })   → topAreaData (요약 카드용)
  ② getWeekly({ fromDate: wed, toDate: tue })        → weeklyData (주별 테이블용)

검색 버튼 클릭:
  ① getWeekly({ fromDate, toDate, suppId })          → topAreaData + weeklyData 동시 갱신
```

---

### 6-3. `DashboardFilterSection`

접기/펼치기 필터 패널. `isOpen` prop으로 토글.

**그리드 레이아웃 (CSS Grid)**

| 조건 | 컬럼 구성 |
|---|---|
| 본사 + 실시간 | 협력사 \| 완료건수 \| 수락률 \| 버튼 (1fr 1fr 1fr auto) |
| 본사 + 주별 | 협력사 \| 주차 \| 완료건수 \| 수락률 \| 버튼 (1fr 1fr 1fr 1fr auto) |
| 협력사 + 실시간 | 완료건수 \| 수락률 \| 버튼 (1fr 1fr auto) |
| 협력사 + 주별 | 주차 \| 완료건수 \| 수락률 \| 버튼 (1fr 1fr 1fr auto) |

**활성 필터 배지:** 우측 상단에 필터 조건이 1개 이상 활성화되면 개수 표시.

---

### 6-4. `DashboardSummaryCards`

4개의 KPI 카드 고정 표시.

| 카드 | 실시간 기준 | 주별 기준 |
|---|---|---|
| 활성 협력사 | 완료건수 ≥ 1 AND 라이더 ≥ 1인 협력사 수 | `activeSupplierCount` |
| 총 완료건수 | 해당일 `dailySlotCounts` 합산 | `totalFinishCnt` |
| 수락률 | `periodAcceptanceRate` | `periodAcceptanceRate` |
| 슬롯달성률 | `periodSlotAchievementRate` | `periodSlotAchievementRate` |

---

### 6-5. `WeeklyPerformanceTable`

**컬럼 구성:** 협력사명 + 날짜 컬럼 (최대 7개, weekStartDate ~ weekEndDate)

**표시 모드**
- **건수 모드:** 각 셀에 "완료/목표" 형식 (예: `15/20`)
- **수행률 모드:** 각 셀에 일별 달성률 % (예: `75%`)

**셀 색상**
| 조건 | 배경색 |
|---|---|
| 달성률 100% 이상 | 초록 계열 (`--vroong-green` 연색) |
| 달성률 1%~99% | 노랑 계열 |
| 달성률 0% | 흰색 |
| N/A | 회색 |

**툴팁 (Portal 렌더링, z-index: 99999)**

셀 hover 시 슬롯별 상세 정보 표시:

```
[슬롯명] [시간대]
완료: X건 / 목표: Y건
달성 여부: 달성 / 미달성 / 목표없음 / 해당없음
```

---

### 6-6. `RealtimePerformanceTable`

**컬럼 구성**

| 컬럼 | 설명 | 정렬 |
|---|---|---|
| 협력사명 | `suppName` | — |
| 셋 수 | `slotSetCnt` | — |
| 완료건수 | `dailySlotCounts` 합산 | 가능 (3단계) |
| 수락률 | `acceptanceRate` | 가능 (3단계) |
| 거절/취소 | `rejectCnt + assignCancelCnt + deliCancelCnt` | 가능 (3단계) |
| 시간대별 수행 | 슬롯 4개 인라인 표시 | — |

**슬롯 인라인 표시 형식:**

```
[슬롯명] [시간대] | X/Y건 | Z%
```

슬롯별 색상 (8가지 프리셋, 슬롯 인덱스 기준):
- 0: 오렌지 (아침점심피크)
- 1: 블루 (오후논피크)
- 2: 보라 (저녁피크)
- 3: 인디고 (심야논피크)

---

### 6-7. `Sidebar`

역할 기반 3단계 메뉴 트리. `navigation.config.ts`의 `NAV_ITEMS`를 기반으로 렌더링.

**메뉴 접근 제어**

| 조건 | 처리 |
|---|---|
| USER_TYPE 미충족 | 메뉴 항목 자체를 렌더링하지 않음 |
| 점검 모드 (협력사) | 대시보드 외 모든 항목 `.disabled` 처리 |
| ACCESS_PAGES 커스텀 권한 | 해당 페이지 항목만 표시 |
| SUPP_TYPE = 'D' (직영) | 일부 정산 메뉴 제외 |
| SUPP_TYPE = 'F' (가맹) | 수수료 관련 메뉴 별도 표시 |

**메뉴 구조 (최상위 7개)**

```
1. 대시보드
2. 공지사항
3. 라이더 (현황 · 배달내역 · 상세내역)
4. 협력사 (현황 · 수수료 관리 [3단계])
5. 정산 (작업현황 · 정산내역 · 공제 · 프로모션)
6. 시스템 관리 [본사 전용] (업로드 · 유저관리 · 회계 등)
7. 정책 관리 [본사 전용] (슬롯 · 휴일 · 표준구역)
```

---

### 6-8. `Header`

**표시 요소**

| 요소 | 조건 |
|---|---|
| 잔액 표시 | USER_TYPE = 'SUPPLIER'일 때만 표시 |
| 유저 이름 / 이메일 | 항상 표시 |
| 로그아웃 드롭다운 | 유저 영역 클릭 시 표시 |

**잔액 조회:** 컴포넌트 마운트 시 `balanceApi.getBalance()` 호출. 실패해도 무시.

---

### 6-9. `DashboardTabBar`

**탭 동작 규칙**

| 규칙 | 설명 |
|---|---|
| 대시보드 탭 | 닫기 버튼 없음 (항상 유지) |
| 중복 탭 | 같은 페이지 열기 시 기존 탭으로 전환 |
| 점검 모드 | 협력사 유저 점검 중 탭 클릭 비활성화 |
| 전체 닫기 | 2개 이상 탭이 있을 때 "전체 닫기" 버튼 표시 |

---

### 6-10. `DashboardTabRouter`

`currentPage` 값에 따라 동적으로 해당 컴포넌트를 렌더링. 26개 이상의 페이지를 지원.

**라우팅 대상 (주요 페이지)**

| currentPage 값 | 컴포넌트 |
|---|---|
| `dashboard` | DashboardMainContent |
| `rider_status` | RiderStatusPage |
| `delivery_history` | DeliveryHistoryPage |
| `delivery_history_detail` | DeliveryHistoryDetailPage |
| `partner_status` | SupplierManagement |
| `partner_fee` | PartnerFeeManagement |
| `settlement_history` | SettlementHistoryPage |
| `deduction_*` | DeductionPage (3종) |
| `promotion_*` | PromotionPage (다수) |
| `system_*` | SystemManagement (다수) |
| `policy_*` | PolicyPage (슬롯 · 휴일 · 구역) |

---

## 7. 비즈니스 로직

### 7-1. 날짜 기준

**VROONG 배달일 기준:** 06:00 ~ 익일 05:59

- `getDashboardDate()`: 현재 시각이 06시 미만이면 전일 날짜 반환
- 주간 기준: **수요일 ~ 화요일** (7일)
- 기본 주차 계산:
  ```
  오늘 → 가장 가까운 수요일(과거) 찾기 → 그 수요일 + 6일 = 화요일
  ```

---

### 7-2. 필터 로직

**완료건수 이상 필터**

```
대상 필드: dailySlotCounts[날짜][슬롯인덱스] = "완료/목표"
계산: 슬롯별 완료 건수(분자) 합산
  - 실시간: selectedDate 기준 단일 날짜 합산
  - 주별: 기간 내 모든 날짜 합산
조건: 합산값 >= 입력값인 협력사만 표시
```

**수락률 이상 필터**

```
대상 필드: acceptanceRate (0~100)
조건: acceptanceRate >= 입력값인 협력사만 표시
```

두 필터는 AND 조건으로 동시 적용.

---

### 7-3. 정렬 로직 (실시간 모드)

3단계 사이클: `없음 → 오름차순 → 내림차순 → 없음`

| 정렬 기준 | 계산 방법 |
|---|---|
| 완료건수 | `dailySlotCounts[selectedDate]`의 분자 합산 |
| 수락률 | `acceptanceRate` 직접 비교 |
| 거절/취소건수 | `rejectCnt + assignCancelCnt + deliCancelCnt` 합산 |

---

### 7-4. 슬롯 데이터 파싱

슬롯 데이터는 두 가지 형식으로 제공:

```
dailySlots[YYYYMMDD][i]      = "75%"       → 달성률 (rate mode)
dailySlotCounts[YYYYMMDD][i] = "15/20"     → 완료/목표 건수 (count mode)
```

유효하지 않은 슬롯 (N/A):
- `"—"`, `"-"`, `""` 등의 값은 해당없음 처리
- `"%"` 를 포함하지 않으면 달성률 없음으로 처리

미션 모드: `dailyMissionRate[YYYYMMDD]` 존재 시 일별 달성률 계산을 override.

---

### 7-5. 요약 카드 계산

**활성 협력사 수**
- 실시간: `finishCnt >= 1 AND riderCount >= 1`인 협력사 수 (클라이언트 계산)
- 주별: API 응답의 `activeSupplierCount` 직접 사용

**총 완료건수**
- 실시간: 모든 협력사의 `dailySlotCounts[selectedDate]` 분자 합산
- 주별: API 응답의 `totalFinishCnt` 직접 사용

---

## 8. 상태 관리 및 데이터 흐름

### localStorage 키 목록

| 키 | 설명 |
|---|---|
| `USER_TYPE` | HEADQUARTERS / SUPPLIER / RIDER |
| `USER_ID` | 사용자 ID (협력사 유저: 협력사 ID와 동일) |
| `USER_NAME` | 사용자 이름 |
| `SUPP_TYPE` | 협력사 유형 (D: 직영, F: 가맹) |
| `PRM_CODE` | 권한 코드 |
| `ACCESS_PAGES` | 커스텀 접근 가능 페이지 목록 |
| `CURRENT_PAGE` | 마지막 활성 페이지 ID |
| `DASHBOARD_TABS` | 열려 있는 탭 배열 (JSON) |
| `ACTIVE_TAB_ID` | 마지막 활성 탭 ID |
| `NOTICE_HIDDEN_TODAY` | 오늘 하루 숨기기 한 공지 ID 목록 (날짜 키) |

### sessionStorage 키 목록

| 키 | 설명 |
|---|---|
| `maintenance_acknowledged` | 점검 화면 확인 여부 |
| `maintenance_acknowledged_seqno` | 확인한 점검 공지 일련번호 |

### 데이터 흐름

```
1. DashboardPage 마운트
   → authApi.getUserInfo() → userInfo 세팅
   → noticeApi.getUnreadPopupNotices() → 팝업 공지 세팅
   → localStorage 복원 (탭, 페이지 상태)

2. DashboardMainContent 마운트
   → supplierApi.getSupplierList() [HQ만] → suppliers 세팅
   → initializeDefaultDateRange() → fromDate/toDate 세팅
   → fromDate/toDate 세팅 완료 useEffect 트리거
     → dashboardApi.getWeekly(today) → topAreaData
     → dashboardApi.getWeekly(fromDate~toDate) → weeklyData

3. 뷰 모드 변경 (실시간 ↔ 주별)
   → 필터 초기화 (supplierId, slotCount, acceptRate)
   → 실시간: selectedDate = 오늘
   → 새 날짜/기간으로 API 재호출

4. 검색 버튼 클릭
   → dashboardApi.getWeekly(fromDate, toDate, suppId) → topAreaData + weeklyData

5. 필터 변경 (슬롯건수/수락률)
   → 클라이언트 사이드 필터링 (API 재호출 없음)

6. 정렬 변경 (실시간 모드)
   → 클라이언트 사이드 정렬 (API 재호출 없음)
```

---

## 9. 설정 파일

### `navigation.config.ts`

사이드바 메뉴 전체 트리를 정의. 각 항목의 구조:

```typescript
{
  id: string;              // 메뉴 고유 ID
  label: string;           // 표시 라벨
  icon: string;            // Font Awesome 아이콘 클래스
  allowedUserTypes?: string[];   // 허용 USER_TYPE 배열
  suppTypeOnly?: string;         // 특정 SUPP_TYPE만 표시
  requirePrmCode?: string;       // 필요 권한 코드
  disabled?: boolean;            // 점검 등으로 비활성화
  submenu?: NavSubItem[];        // 2단계 하위 메뉴
  children?: NavChildItem[];     // 3단계 하위 메뉴
  page?: string;                 // 연결 페이지 ID
}
```

---

### `filterKeys.config.ts`

탭 닫을 때 해당 페이지의 localStorage 필터 상태를 정리하기 위한 매핑.

```typescript
// 예시
{
  'partner_status': ['supplierSelectionFilters'],
  'rider_status': ['riderStatusFilters'],
  'delivery_history': ['deliveryHistoryFilters'],
  // ... 20개 이상
}
```

---

### `pageLabels.config.ts`

`currentPage` 값 → 탭 표시 라벨 매핑. 40개 이상의 페이지를 커버.

```typescript
{
  'dashboard': '대시보드',
  'rider_status': '라이더 현황',
  'partner_status': '협력사 현황',
  'common_code_add': '공통코드 추가',    // 추가/수정 모드에 따라 동적 변경
  // ...
}
```

---

## 10. 유틸리티 함수

### `dashboardUtils.ts`

| 함수 | 반환 | 설명 |
|---|---|---|
| `formatNumber(value)` | string | 숫자를 로케일 형식으로 포맷 (실패 시 `"-"`) |
| `getWeekdayOnly(yyyymmdd)` | string | `"20241218"` → `"수"` (요일 한글 1글자) |
| `getDashboardDate()` | string | 오늘 날짜 (06시 미만이면 전일) YYYY-MM-DD |
| `generateWeekDates(start, end)` | string[] | 기간 내 날짜 배열 (YYYYMMDD 형식) |
| `getSlotTooltipItems(supplier, date)` | object[] | 해당 날짜 슬롯별 툴팁 표시용 데이터 |

**상수**

| 상수 | 설명 |
|---|---|
| `SLOT_COLORS` | 슬롯별 색상 객체 배열 8개 (`headerBg`, `headerText`, `rateText`) |
| `DEFAULT_SLOT_NAMES` | `['아침점심피크', '오후논피크', '저녁피크', '심야논피크']` |

---

### `noticeStorageUtils.ts`

| 함수 | 설명 |
|---|---|
| `getHiddenNoticeIdsForToday()` | 오늘 날짜 기준 숨겨진 공지 ID 배열 반환 |
| `saveHiddenNoticeForToday(id)` | 공지 ID를 오늘 날짜에 묶어 저장 |

날짜가 바뀌면 이전 날짜의 숨기기 기록은 자동 무효화.

---

## 11. CSS 구조

### 파일별 역할

| 파일 | 크기 | 주요 담당 |
|---|---|---|
| `DashboardPage.css` | ~1100줄 | 전체 레이아웃, 테이블, 필터, 카드, 툴팁, 반응형 |
| `Header.css` | ~150줄 | 헤더 전용 스타일 (공통 CSS 외 추가/재정의) |
| `Sidebar.css` | ~155줄 | 사이드바 전용 스타일 (공통 CSS 외 추가/재정의) |
| `Footer.css` | ~30줄 | 하단 고정 푸터 |

**공통 CSS와의 관계**
- `src/styles/layout.css`: 공통 레이아웃 클래스 정의 (`.sidebar`, `.header`, `.nav-link` 등)
- `src/styles/vroong-design-system.css`: CSS 변수 정의 (`--vroong-green`, `--spacing-*` 등)
- 컴포넌트 CSS는 공통 CSS를 import 후 추가/재정의하는 구조
- 중복 정의는 제거하고 의도적 재정의는 주석으로 이유 명시

### 주요 CSS 변수

| 변수 | 값 | 사용처 |
|---|---|---|
| `--vroong-green` | `#05CD82` | 버튼, 활성 상태, 달성 색상 |
| `--color-gray-*` | 50~900 | 텍스트, 배경, 보더 |
| `--font-size-*` | xs~xl | 텍스트 크기 |
| `--spacing-*` | 1~12 | 여백 |
| `--border-radius-*` | sm~xl | 모서리 |
| `--shadow-*` | sm~xl | 그림자 |

### 반응형 브레이크포인트

| 브레이크포인트 | 주요 변화 |
|---|---|
| `max-width: 1400px` | 요약 카드 2열 → 2열, 필터 그리드 축소 |
| `max-width: 1024px` | 헤더 컨트롤 컬럼 방향 전환 |
| `max-width: 768px` | 사이드바 오프스크린, 1열 그리드, 모바일 메뉴 버튼 표시 |

---

## 12. 권한 및 역할별 차이

### 화면 요소 차이

| 요소 | HEADQUARTERS | SUPPLIER | RIDER |
|---|---|---|---|
| 협력사 선택 필터 | 표시 | 미표시 | 해당 없음 |
| 잔액 표시 (헤더) | 미표시 | 표시 | 미표시 |
| 사이드바 메뉴 | 전체 | 제한된 메뉴 | 제한된 메뉴 |
| 시스템 관리 메뉴 | 표시 | 미표시 | 미표시 |
| 정책 관리 메뉴 | 표시 | 미표시 | 미표시 |
| 정산 메뉴 | 표시 | 일부 (SUPP_TYPE 기준) | 미표시 |

### 데이터 범위 차이

| 유형 | getWeekly suppId | 결과 범위 |
|---|---|---|
| 본사 (전체) | 미전송 | 전체 협력사 |
| 본사 (필터 선택) | 선택한 suppId | 해당 협력사만 |
| 협력사 | 본인 USER_ID | 본인 데이터만 |

### 점검 모드 동작

| 유형 | 점검 모드 시 |
|---|---|
| HEADQUARTERS | 점검 화면 미표시, 정상 운영 |
| SUPPLIER | 점검 화면 표시, 확인 후 대시보드만 접근 가능 |

---

*이 문서는 `src/features/dashboard/` 소스코드를 기반으로 작성되었습니다.*  
*코드 변경 시 해당 섹션을 함께 업데이트해주세요.*
