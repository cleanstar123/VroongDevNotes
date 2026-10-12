# vroong-backend 로컬 실행 필수 준비

## 사전 요구사항

| 항목 | 필요 여부 | 비고 |
|------|----------|------|
| Docker | **필수** | Redis 실행용 |
| Java (JDK) | **필수** | Spring Boot 실행용 |
| AWS RDS 접근 | **필수** | 보안 그룹 IP 허용 필요 |
| relay-server | 선택 | 배민 크롤링 기능 사용 시만 |

---

## 실행 순서

### 1. Redis 실행 (Docker)

```bash
cd C:\wonjun\vroong-backend\scripts
docker-compose up -d redis
```

- 처음 실행 시 Docker가 자동으로 이미지 pull → 컨테이너 생성 → 시작
- 이후 Docker Desktop을 켜면 `vroong-redis` 컨테이너 자동 재시작 (`restart: unless-stopped`)

확인:
```bash
docker ps  # vroong-redis 컨테이너가 running 상태인지 확인
```

### 2. AWS RDS 보안 그룹 IP 허용

`application-local.yml` DB가 AWS RDS를 바라봄:
```
vraccountdb.c1acmo8sqyvw.ap-northeast-2.rds.amazonaws.com:3306
```

새로운 로컬 환경에서 실행할 경우 AWS 콘솔 → RDS 보안 그룹 → 인바운드 규칙에 본인 IP(포트 3306) 추가 필요.
허용되지 않으면 기동 시 DB 연결 실패 오류 발생.

### 3. Spring Boot 실행

**터미널:**
```bash
cd C:\wonjun\vroong-backend
gradlew :api:bootRun --args='--spring.profiles.active=local'
```

**IntelliJ Run Configuration:**
- Active profiles: `local`
- 별도 환경변수 불필요 (`application-local.yml`에 기본값 모두 설정되어 있음)

---

## 주요 설정 위치

| 설정 | 파일 |
|------|------|
| 로컬 프로파일 설정 | `api/src/main/resources/application-local.yml` |
| 공통 설정 (Redis 등) | `api/src/main/resources/application.yml` |
| Docker 설정 | `scripts/docker-compose.yml` |

---

## 기본 포트

| 서비스 | 포트 |
|--------|------|
| Spring Boot API | 8080 |
| Redis | 6379 |
| relay-server | 8090 |
