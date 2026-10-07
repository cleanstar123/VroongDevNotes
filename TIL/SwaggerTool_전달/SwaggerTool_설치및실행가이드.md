# VROONG Swagger Tool 설치 및 실행 가이드

GitHub 저장소: https://github.com/umacitc/vroong-3pl-platform-tool.git

---

## 사전 요구사항

- **Node.js 18 이상** 설치 필요 → https://nodejs.org

---

## 방법 1: 설치 파일(.exe) 빌드 후 사용 (권장)

```bash
# 1. 프로젝트 클론
git clone https://github.com/umacitc/vroong-3pl-platform-tool.git
cd vroong-3pl-platform-tool

# 2. 패키지 설치
npm install

# 3. Windows 설치 파일 생성
npm run package:win
```

빌드 완료 후 `release/VROONG Swagger Tool Setup 1.0.1.exe` 를 실행하여 설치합니다.

---

## 방법 2: 개발 모드로 바로 실행

```bash
# 1. 프로젝트 클론
git clone https://github.com/umacitc/vroong-3pl-platform-tool.git
cd vroong-3pl-platform-tool

# 2. 패키지 설치
npm install

# 3. 개발 모드 실행
npm run dev
```

---

## 배달현황 배치 기능 추가 설정

배달현황 배치 기능을 사용하려면 아래 두 가지가 필요합니다.

### 1. AWS CLI 설치 및 인증

ECS 릴레이 서버 세션 상태 확인에 사용됩니다.

```bash
aws configure
# AWS Access Key ID, Secret Access Key, Region (ap-northeast-2) 입력
```

### 2. 릴레이 서버 경로 수정

`src/main/index.ts` 상단의 경로를 본인 PC에 맞게 수정합니다.

```typescript
// 변경 전 (기본값)
const RELAY_SERVER_PATH = 'C:\\wonjun\\vroong-backend\\relay-server';

// 변경 후 (본인 PC 경로로 수정)
const RELAY_SERVER_PATH = 'C:\\본인경로\\vroong-backend\\relay-server';
```

경로 수정 후 다시 빌드해야 적용됩니다.

```bash
npm run package:win
```

---

## 주요 스크립트 정리

| 명령어 | 설명 |
|---|---|
| `npm install` | 패키지 설치 |
| `npm run dev` | 개발 모드 실행 |
| `npm run package:win` | Windows 설치 파일(.exe) 생성 |

---

## 기술 스택

- React 18 + TypeScript
- Electron 28
- Ant Design 5
- Vite
