# Spring Boot Redis 세션 클러스터링 + Jenkins CI/CD 스터디

Redis를 공유 세션 저장소로 사용하여 멀티 인스턴스 환경에서 세션 일관성을 유지하는 방법과, Jenkins 파이프라인을 통한 자동 배포를 실습한 스터디 프로젝트입니다.

---

## 아키텍처

```
Client
  │
  ▼
Nginx (:80)          ← 로드밸런서 (라운드로빈)
  ├── app-1 (:8080)  ─┐
  └── app-2 (:8080)  ─┴─► Redis (:6379)  ← 공유 세션 저장소
  └── /jenkins/      ─► Jenkins (:8080)  ← CI/CD (리버스 프록시)

Jenkins (:9090)  ← 직접 접근용 (초기 설정 시)
```

> AWS 환경: `EC2 → ALB → Route53 도메인`으로 외부 접근

---

## 로컬 실행

### 방법 A — IntelliJ 직접 실행 (Redis만 Docker)

Redis를 로컬 Docker로 먼저 실행한 뒤, IntelliJ에서 `local` 프로파일로 앱을 시작합니다.

```bash
# Redis 컨테이너 실행
docker run -d --name redis-local -p 6379:6379 redis:latest
```

### 방법 B — 전체 스택 Docker Compose

```bash
# 빌드
./gradlew build -x test --no-daemon

# 전체 스택 실행 (Redis + app-1 + app-2 + Nginx + Jenkins)
docker compose -p springboot-redis up --build -d

# 확인
curl http://localhost/
```

브라우저에서 `http://localhost/` 반복 접속 시, 응답 서버(app-1 ↔ app-2)가 교차되지만 세션 최초 접속 서버는 유지됩니다.

---

## AWS 배포

### 인프라 구성 순서

| 단계 | 항목 | 비고 |
|------|------|------|
| 1 | EC2 생성 | Amazon Linux 2023, ARM(t4g.small), 퍼블릭 IP 자동 할당 |
| 2 | IAM 역할 부여 | `AmazonSSMManagedInstanceCore` 포함 → Session Manager 사용 가능 |
| 3 | 보안그룹 설정 | 인바운드: HTTP 80(ALB), 아웃바운드: All traffic |
| 4 | Docker 설치 | ARM64 기준 수동 설치 필요 (아래 참고) |
| 5 | ALB 설정 | 대상그룹 포트 80, 헬스체크 경로 `/` |
| 6 | ALB 보안그룹 | **아웃바운드 All traffic 반드시 추가** |
| 7 | Route53 | A 레코드 → ALB Alias |

### EC2 Docker 설치 (Amazon Linux 2023 ARM64)

```bash
# Docker Engine
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

# docker compose 플러그인 (수동)
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/download/v2.36.0/docker-compose-linux-aarch64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# buildx (수동)
sudo curl -SL https://github.com/docker/buildx/releases/download/v0.21.2/buildx-v0.21.2.linux-arm64 \
  -o /usr/local/lib/docker/cli-plugins/docker-buildx
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx
```

---

## Jenkins CI/CD

### 파이프라인 구성 (Jenkinsfile)

```
Git Clone → Gradle Build → Docker Build & Deploy → Health Check
```

### Jenkins 접근

| 방법 | URL | 용도 |
|------|-----|------|
| 직접 접근 | `http://{EC2-IP}:9090` | **초기 설정 전용** (EC2 인바운드 9090 임시 오픈) |
| Nginx 프록시 | `https://{도메인}/jenkins/` | 초기 설정 완료 후 정상 사용 |

> 초기 설정(Unlock Jenkins, 플러그인 설치)은 반드시 직접 포트(9090)로 접근해야 합니다.  
> Nginx 리버스 프록시로는 CSRF 보호로 인해 403 오류가 발생할 수 있습니다.

### Jenkins 내 Git 연동

- Freestyle 또는 Pipeline 프로젝트 생성
- SCM: Git, Repository URL, Branch: `*/main`
- Build Triggers: GitHub hook 또는 수동 빌드

---

## 핵심 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| 로컬 실행 시 Redis 연결 실패 | Docker 컨테이너용 호스트명(`redis-server`) 사용 | Spring 프로파일로 Redis host를 `localhost`로 오버라이드 |
| Session Manager 입력 불가 | 퍼블릭 IP 없음 또는 IAM 역할 미부여 | 퍼블릭 IP 자동 할당 + IAM 역할에 SSM 정책 추가 |
| ALB 504 Gateway Timeout | ALB 보안그룹 아웃바운드 규칙 없음 | ALB 보안그룹에 아웃바운드 All traffic 추가 |
| Jenkins 초기 설정 403 | Nginx X-Forwarded-Proto 헤더 미설정 | 직접 포트(9090)로 초기 설정 진행 |
| Nginx 설정 미반영 | `nginx -s reload` 미작동 | `docker compose restart nginx` |
| Jenkins 빌드 중 인스턴스 응답 없음 | t4g.small 메모리(2GB) 부족 | 인스턴스 재부팅 또는 상위 스펙 사용 |

---

## 프로젝트 구조

```
├── src/main/
│   ├── java/.../
│   │   ├── SpringbootRedisApplication.java
│   │   └── SessionController.java       # GET /, 세션 클러스터링 시연
│   └── resources/
│       └── application.yml              # Redis host: redis-server (Docker용)
├── nginx/nginx.conf                     # 로드밸런싱 + Jenkins 리버스 프록시
├── docker-compose.yml                   # 전체 스택 정의
├── Dockerfile                           # eclipse-temurin:21 멀티스테이지 빌드
├── Jenkinsfile                          # CI/CD 파이프라인 4단계
└── .gitattributes                       # LF 강제 적용 (Linux 배포 호환)
```

---

## 주요 의존성

```
spring-boot-starter-web
spring-boot-starter-data-redis
spring-session-data-redis
```
