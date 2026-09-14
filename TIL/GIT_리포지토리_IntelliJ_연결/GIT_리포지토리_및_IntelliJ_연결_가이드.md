# GIT 리포지토리 정보 및 IntelliJ 연결 가이드

> 작성일: 2026-09-10

---

## 목차

1. [리포지토리 목록](#1-리포지토리-목록)
2. [사전 준비](#2-사전-준비)
3. [IntelliJ에서 리포지토리 클론 및 연결](#3-intellij에서-리포지토리-클론-및-연결)
4. [prod / dev 브랜치 코드 내려받기](#4-prod--dev-브랜치-코드-내려받기)
5. [브랜치 전략 요약](#5-브랜치-전략-요약)

---

## 1. 리포지토리 목록

| 번호 | 리포지토리명 | 역할 | HTTPS |
|------|-------------|------|-------|
| 1 | `vroong-3pl-platform-front` | 웹 프론트엔드 | `https://github.com/umacitc/vroong-3pl-platform-front.git` |
| 2 | `vroong-3pl-platform-back` | 백엔드 API 서버 | `https://github.com/umacitc/vroong-3pl-platform-back.git` |
| 3 | `vroong-3pl-platform-app` | 모바일 앱 (React + Capacitor) | `https://github.com/umacitc/vroong-3pl-platform-app.git` |
| 4 | `vroong-3pl-platform-batch` | 배치 작업 서버 | `https://github.com/umacitc/vroong-3pl-platform-batch.git` |

---

## 2. 사전 준비

### Git 설치 확인

IntelliJ 터미널 또는 PowerShell에서:

```powershell
git --version
# git version 2.x.x 가 출력되면 정상
```

미설치 시: https://git-scm.com/downloads 에서 설치

### GitHub 계정 인증 설정

HTTPS 방식으로 클론할 경우 GitHub 계정 인증이 필요합니다.

```powershell
git config --global user.name "깃유저네임"
git config --global user.email "깃이메일"
```

> GitHub에서 2FA(2단계 인증)를 사용 중이라면 비밀번호 대신 **Personal Access Token(PAT)** 을 사용해야 합니다.
> GitHub → Settings → Developer settings → Personal access tokens → Generate new token

---

## 3. IntelliJ에서 리포지토리 클론 및 연결

### 방법 A: IntelliJ 시작 화면에서 클론 (신규)

```
1. IntelliJ 실행
2. 시작 화면 → [Get from VCS] 클릭
3. URL 입력란에 HTTPS 주소 붙여넣기
   예) https://github.com/umacitc/vroong-3pl-platform-back.git
4. Directory: 로컬 저장 경로 설정
   예) C:\wonjun\vroong-3pl-platform-back
5. [Clone] 클릭
6. GitHub 로그인 창이 뜨면 계정 인증 진행
```

### 방법 B: 이미 열린 프로젝트에서 클론

```
상단 메뉴 → Git → Clone
→ URL 입력 → Directory 설정 → [Clone] 클릭
```

### 방법 C: 로컬에 이미 폴더가 있는 경우 (기존 폴더 연결)

```
상단 메뉴 → File → Open
→ 해당 프로젝트 폴더 선택 → [OK]
→ 하단 Git 탭 또는 우측 하단에 브랜치 이름이 표시되면 연결 완료
```

### 클론 완료 확인

IntelliJ 우측 하단에 현재 브랜치 이름이 표시되면 Git 연결 성공입니다.

```
예) main  또는  prod  또는  dev
```

---

## 4. prod / dev 브랜치 코드 내려받기

### 방법 A: IntelliJ UI로 브랜치 변경 및 pull

```
1. IntelliJ 우측 하단 브랜치 이름 클릭
2. [Remote Branches] 목록에서 origin/prod 또는 origin/dev 선택
3. [Checkout] 클릭 → 해당 브랜치로 전환
4. 상단 메뉴 → Git → Pull (또는 단축키 Ctrl+T)
   → 최신 코드 내려받기 완료
```

### 방법 B: IntelliJ 터미널에서 명령어로

IntelliJ 하단 Terminal 탭을 열고 직접 입력:

```bash
# 원격 브랜치 목록 최신화
git fetch --all

# prod 브랜치로 전환 후 최신 코드 받기
git checkout prod
git pull origin prod

# dev 브랜치로 전환 후 최신 코드 받기
git checkout dev
git pull origin dev
```

### 최초 클론 시 (브랜치가 로컬에 없는 경우)

```bash
# 원격의 prod 브랜치를 로컬에 생성하며 전환
git checkout -b prod origin/prod

# 원격의 dev 브랜치를 로컬에 생성하며 전환
git checkout -b dev origin/dev
```

### 현재 상태 확인

```bash
# 현재 브랜치 및 로컬/원격 브랜치 목록 확인
git branch -a

# 현재 브랜치의 최신 커밋 확인
git log --oneline -5
```

---

## 5. 브랜치 전략 요약

| 브랜치 | 용도 | 설명 |
|--------|------|------|
| `main` | 기준 브랜치 | PR 기반으로 관리 |
| `prod` | 운영 배포 브랜치 | 실제 서비스에 배포되는 코드 |
| `dev` | 개발 브랜치 | 개발 중인 기능, 테스트 환경 배포용 |

### 코드 흐름

```
기능 개발 (feature 브랜치)
      ↓
    dev 병합 → 개발 서버 배포 및 테스트
      ↓
   prod 병합 → 운영 서버 배포
```

> **주의:** prod 브랜치에 직접 커밋하지 말고 반드시 dev에서 검증 후 병합합니다.
