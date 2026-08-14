# 대시보드 브랜치 현황 비교 (prod vs dev)

> **작성일:** 2026-08-14  
> **목적:** prod 브랜치 현재 상태 분석 및 dev 브랜치 리팩토링 결과와의 차이 정리

---

## 브랜치 관계 요약

```
prod  ──● 1a556c4  대시보드 주별 > 표 슬롯 달성 % 버그 수정
        ● 8b2db0e  배달 취소/건수 버그 수정
        ● e174498  완료 건수 내림차순/올림차순 버그 수정
        ● 8917a33  거절/취소 건수 버그 수정
        ● ...
        
dev   ──● d7aa88c  features/dashboard 리팩토링 v1   ← 컴포넌트 분리 + CSS 정리
        ● cceba17  버그수정 4개 합본 prod->dev
        ● 1e2a44d  시간대 변경 & CSS UI 수정 prod->dev
        ● ...
```

- `dev`에만 있는 커밋: 리팩토링 커밋 포함 다수
- `prod`에만 있는 커밋: prod에서 직접 작성한 버그 수정 커밋들 (dev로 머지 필요)

---

## 1. 파일 구조 차이

### prod 브랜치 (현재)

```
components/pages/DashboardPage/
├── DashboardPage.tsx         (703줄)  ← 메인 컨테이너 + 페이지 라우팅 포함
└── DashboardMainContent.tsx  (881줄)  ← 수행현황 뷰 전체 + 테이블 + 툴팁 포함
```

### dev 브랜치 (리팩토링 후)

```
components/pages/DashboardPage/
├── DashboardPage.tsx             (더 슬림)
├── DashboardTabBar.tsx           ← 탭 바 분리
├── DashboardTabRouter.tsx        ← 페이지 라우팅 분리
├── DashboardMainContent.tsx      (350줄 수준으로 축소)
├── DashboardFilterSection.tsx    ← 필터 패널 분리
├── DashboardSummaryCards.tsx     ← 요약 카드 분리
├── WeeklyPerformanceTable.tsx    ← 주별 테이블 분리
└── RealtimePerformanceTable.tsx  ← 실시간 테이블 분리
```

**요약:** prod의 2개 파일이 dev에서 8개 파일로 분리됨 (단일 책임 원칙 적용)

---

## 2. prod 브랜치 파일별 상세 현황

### 2-1. `DashboardPage.tsx` (703줄)

**역할:** 레이아웃 컨테이너 + 탭 관리 + 페이지 라우팅 + 사용자 정보 + 공지 팝업 + 점검 모드

**문제점:**
- 단일 컴포넌트에 너무 많은 역할 혼재 (SRP 위반)
- `renderPageContent()` 내부에 50개 이상의 조건부 렌더링 case문
- `useEffect` 9개 이상 (서로 다른 관심사를 한 컴포넌트에서 관리)
- 점검 모드 로직이 Sidebar.tsx와 중복

**주요 상태:**

| 상태 | 설명 |
|---|---|
| `tabs`, `activeTabId`, `currentPage` | 탭 관리 |
| `userInfo`, `isUserInfoLoading` | 사용자 정보 |
| `popupNotices`, `visibleNoticeIds`, `noticeZIndexMap`, `maxZIndex` | 팝업 공지 |
| `maintenanceAcknowledged`, `isMaintenance`, `activeMaintenance` | 점검 모드 |

---

### 2-2. `DashboardMainContent.tsx` (881줄)

**역할:** 협력사별 수행현황 뷰 전체 (필터, 요약 카드, 주별/실시간 테이블, 툴팁까지 포함)

**문제점:**
- 881줄 단일 컴포넌트 (테이블 렌더링, 툴팁 로직, API 호출, 필터/정렬 모두 혼재)
- Portal 기반 툴팁 위치 계산 로직(`getBoundingClientRect`) 인라인 처리
- `formatNumber`, `getWeekdayOnly` 등 유틸 함수가 추출되지 않고 컴포넌트 내부에 존재
- `DEFAULT_SLOT_NAMES`, `SLOT_COLORS` 상수 하드코딩

**주요 상태 (dev 대비 추가된 상태):**

| 상태 | 설명 |
|---|---|
| `hoveredCell` | 호버된 셀 정보 (행, 열, 날짜) |
| `tooltipPosition` | 툴팁 위치 (x, y 픽셀) |

> dev 브랜치에서는 툴팁이 `WeeklyPerformanceTable` 컴포넌트 내부로 이동하면서 `DashboardMainContent`에서 제거됨

---

### 2-3. `Header.tsx` (약 150줄)

**prod와 dev의 주요 차이:**

| 항목 | prod | dev |
|---|---|---|
| 로그아웃 처리 | 컴포넌트 내부에서 localStorage 12개 키 직접 정리 | `useAuth` hook + `authApi` 를 통해 처리 |
| 의존성 | `balanceApi`, `useNavigate` | + `useAuth`, `authApi`, `removeItem`, `STORAGE_KEYS` |
| 코드 스타일 | 2-space 들여쓰기, 쌍따옴표 | 4-space 들여쓰기, 홑따옴표 |

---

### 2-4. `Sidebar.tsx` (471줄)

- prod와 dev 모두 복잡도 높음 (471줄 수준 유지)
- 역할 기반 필터링, 3단계 메뉴, 점검 모드 비활성화 로직 포함
- `useMemo`, `useCallback` 적극 사용

**주목할 이슈:**
- 점검 모드 필터링 로직이 `DashboardPage`에도 존재 → 중복

---

### 2-5. CSS 파일 비교

| 파일 | prod 줄수 | dev 줄수 | 차이 |
|---|---|---|---|
| `DashboardPage.css` | 969줄 | 약 1100줄 | dev에서 필터 그리드 · 버튼 스타일 추가 |
| `Header.css` | 206줄 | 약 157줄 | dev에서 layout.css 중복 제거 |
| `Sidebar.css` | 212줄 | 약 155줄 | dev에서 layout.css 중복 제거 |

**prod CSS 주요 문제점:**

1. **layout.css 중복 정의**
   - `Header.css`에 `.header`, `.header-left`, `.header-right`, `.user-menu`, `.user-avatar` → layout.css와 동일
   - `Sidebar.css`에 `.sidebar`, `.sidebar-header`, `.sidebar-nav`, `.nav-group`, `.nav-link:hover`, `.nav-link.active` → layout.css와 동일
   - dev에서 이 중복 정의들을 모두 제거함

2. **하드코딩된 색상값**
   - `#16a34a`, `#065f46`, `#f9fafb`, `#f1f5f9`, `#e2e8f0` 등 CSS 변수 대신 직접 지정
   - `--vroong-green` 변수가 있음에도 일부 구간에서 미사용

3. **DashboardPage.css 내부 중복**
   - `.dashboard-weekly-layout .search-form { grid-template-columns: auto auto auto 1fr }` (옛 규칙)와
     `.dashboard-search-form { grid-template-columns: 1fr 1fr 1fr auto }` (새 규칙)가 공존
   - 옛 규칙(셀렉터 특이도 0,2,0)이 새 규칙(0,1,0)을 무음으로 override하는 버그 존재
   - dev에서 옛 규칙 삭제로 해결

4. **필터 그리드 레이아웃 버그 (prod 미수정)**
   - 주별(weekly) 모드에서 button-group이 다음 줄로 밀리는 현상
   - 완료건수/수락률 form-group의 높이 불일치
   - dev에서 `weekly-mode` 클래스 및 그리드 재정의로 해결

---

## 3. prod 브랜치 주요 알려진 버그 (해결 완료)

prod에서 직접 수정된 버그들 (dev에 머지 필요):

| 커밋 | 버그 내용 |
|---|---|
| `1a556c4` | 주별 테이블 슬롯 달성 % 계산 오류 |
| `8b2db0e` | 배달 취소 건수 표시 오류 |
| `e174498` | 완료 건수 내림차순/오름차순 정렬 오류 |
| `8917a33` | 거절/취소 건수 집계 오류 |
| `6bd19ee` | CSS UI 수정 |
| `de102e6` | 하루 시간대 기준 변경 (06시) |
| `7d9c3b1` | DEFAULT 시간대 버그 수정 |
| `0594e6b` | 총 완료 건수 집계 버그 수정 |

---

## 4. dev 브랜치에서 해결된 사항 (prod 미적용)

### CSS 개선
- `DashboardPage.css` 내부 중복/충돌 셀렉터 제거
- `Header.css` layout.css 중복 제거 (~50줄 감소)
- `Sidebar.css` layout.css 중복 제거 (~57줄 감소)
- 버튼 스타일 현대화 (검색: flat solid green, 초기화: ghost)
- 필터 그리드 레이아웃 버그 수정 (`weekly-mode` 클래스 기반 4단 그리드)
- form-group 높이 불일치 수정

### 컴포넌트 분리
- `DashboardPage.tsx` → `DashboardTabBar` + `DashboardTabRouter` 분리
- `DashboardMainContent.tsx` → `DashboardFilterSection` + `DashboardSummaryCards` + `WeeklyPerformanceTable` + `RealtimePerformanceTable` 분리

---

## 5. prod 브랜치 아키텍처 이슈 (향후 개선 대상)

### 라우팅 구조
- React Router 대신 `currentPage` 상태 + 조건부 렌더링으로 페이지 전환
- 50+ case문 → config 기반 라우팅으로 전환 필요

### 상태 관리
- localStorage를 각 컴포넌트에서 직접 접근 (공통 store 없음)
- 탭 상태 + 사용자 정보 + 점검 모드를 단일 DashboardPage에서 모두 관리
- Context API 또는 Zustand 도입 검토 필요

### API 에러 처리
- 모든 API 호출이 `try/catch` + 빈 catch 블록 (사용자에게 오류 피드백 없음)
- 네트워크 실패 시 빈 화면 또는 이전 데이터 유지

### 코드 스타일 혼재
- prod 파일: 2-space 들여쓰기, 쌍따옴표
- dev 리팩토링 파일: 4-space 들여쓰기, 홑따옴표
- ESLint/Prettier 설정 통일 필요

---

## 6. 권장 작업 순서

```
① prod 버그 수정 커밋들을 dev에 cherry-pick 또는 rebase
② dev 리팩토링 내용 검증 (CSS 중복 제거, 컴포넌트 분리)
③ dev → prod 머지
④ (향후) API 에러 처리 추가
⑤ (향후) 라우팅 구조 React Router 전환 검토
⑥ (향후) 상태 관리 레이어 도입 검토
```

---

*이 문서는 `git diff prod dev -- src/features/dashboard/` 결과 및 각 브랜치 소스 분석을 기반으로 작성되었습니다.*
