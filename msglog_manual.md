# MSGLOG 서버 인프라 통합 매뉴얼

## 📋 목차

1. [시스템 개요](#1-시스템-개요)
2. [아키텍처 구조](#2-아키텍처-구조)
3. [초기 환경 구성](#3-초기-환경-구성)
4. [서비스 배포 절차](#4-서비스-배포-절차)
5. [운영 및 유지보수](#5-운영-및-유지보수)
6. [트러블슈팅](#6-트러블슈팅)

---

## 1. 시스템 개요

### 1.1 서비스 소개

MSGLOG는 Docker 기반의 컨테이너화된 웹 애플리케이션 서비스로, Linux 물리 서버에서 Docker Swarm 모드로 운영됩니다.

### 1.2 기술 스택

| 구성 요소 | 기술 |
|---------|------|
| **운영체제** | Ubuntu Linux 20.04 LTS 이상 |
| **컨테이너** | Docker CE, Docker Swarm |
| **웹 서버** | Nginx (Reverse Proxy & Static Files) |
| **백엔드** | Node.js (MSGLOG API) |
| **프론트엔드** | Static HTML/JS/CSS (Pre-built) |
| **데이터베이스** | MariaDB 11.6.2 |
| **캐시/세션** | Redis (Socket.io Adapter) |
| **TLS 인증서** | Let's Encrypt (Certbot) |

### 1.3 시스템 요구사항

**최소 사양**
- CPU: 2 Core 이상
- RAM: 4GB 이상
- 디스크: 20GB 여유 공간
- 네트워크: 고정 공인 IP 및 도메인

**필수 권한**
- Root 또는 sudo 권한
- Docker 그룹 소속

---

## 2. 아키텍처 구조

### 2.1 시스템 구성도

```
🌐 Internet (User Browser)
        |
        | HTTPS (443)
        ↓
┌─────────────────────────────────────────────┐
│  🖥️  Linux Physical Server                  │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │  Web Gateway Server Container          │ │
│  │  - Nginx (Port 80/443)                 │ │
│  │  - Let's Encrypt (Certbot)             │ │
│  │  - TLS Termination                     │ │
│  └────────────────────────────────────────┘ │
│           |                                  │
│           | Domain Routing                   │
│           ↓                                  │
│  ┌────────────────────────────────────────┐ │
│  │  MSGLOG Web Container (Frontend)       │ │
│  │  - Static Files (HTML/JS/CSS)          │ │
│  │  - Port 3000                           │ │
│  └────────────────────────────────────────┘ │
│           |                                  │
│           | /api requests                    │
│           ↓                                  │
│  ┌────────────────────────────────────────┐ │
│  │  MSGLOG API Container (Backend)        │ │
│  │  - Node.js API Server                  │ │
│  │  - Socket.io Server                    │ │
│  └────────────────────────────────────────┘ │
│           |              |                   │
│           |              |                   │
│     ┌─────┘              └─────┐            │
│     ↓                          ↓            │
│  ┌──────────────┐      ┌──────────────┐    │
│  │  MariaDB     │      │  Redis       │    │
│  │  Container   │      │  Container   │    │
│  │  - Port 3306 │      │  - Port 6379 │    │
│  │  - Data      │      │  - Session   │    │
│  │    Storage   │      │  - Pub/Sub   │    │
│  └──────────────┘      └──────────────┘    │
└─────────────────────────────────────────────┘
```

### 2.2 Docker Stack 구조

MSGLOG 서비스는 2개의 Docker Stack으로 구성됩니다:

#### Stack 1: Gateway Stack (`kim5257_gateway`)

| 서비스 | 컨테이너 | 포트 | 역할 |
|-------|---------|-----|------|
| **nginx** | Web Gateway | 80, 443 | HTTPS 종단, 리버스 프록시, 도메인 라우팅 |
| **redis** | Redis Cache | 6379 | Socket.io 세션 스토어, Pub/Sub |
| **nameserver** | BIND9 DNS | 53 | 내부 DNS (선택사항) |

#### Stack 2: Database Stack (`kim5257_db`)

| 서비스 | 컨테이너 | 포트 | 역할 |
|-------|---------|-----|------|
| **mariadb** | MariaDB | 3306 | 데이터베이스 서버 |

### 2.3 네트워크 구성

**Overlay Network: `backbone`**
- Gateway Stack과 Database Stack을 연결
- 컨테이너 간 내부 통신용
- 서비스 디스커버리 지원

### 2.4 데이터 볼륨 구조

모든 영구 데이터는 호스트의 `/home/$USER/nfs/` 디렉토리에 마운트됩니다.

```
/home/$USER/nfs/
├── kim5257-gateway-nginx/
│   ├── conf.d/                    # Nginx 도메인별 설정
│   │   └── example.com.conf
│   ├── htdocs/
│   │   └── letsencrypt/          # ACME Challenge 디렉토리
│   └── cert/                      # TLS 인증서 저장소
│       └── example.com/
│           ├── fullchain.pem
│           └── privkey.pem
│
├── kim5257-msg-log-prod-nginx/
│   └── htdocs/                    # Frontend 정적 파일
│       ├── index.html
│       └── assets/
│
└── kim5257-db-mariadb/
    └── data/                      # MariaDB 데이터 디렉토리
```

---

## 3. 초기 환경 구성

### 3.1 사전 준비

#### 필수 정보 준비
```bash
# 서버 정보 확인
echo "서버 IP: $(curl -s ifconfig.me)"
echo "사용자명: $USER"
echo "홈 디렉토리: $HOME"

# 도메인 준비 (DNS A 레코드 설정 필요)
# - 메인 도메인: example.com → 서버 IP
# - API 서브도메인: api.example.com → 서버 IP
```

#### 보안 정보 사전 결정
- MariaDB root 비밀번호
- MariaDB 애플리케이션 사용자 비밀번호
- Redis 비밀번호 (권장)

### 3.2 Step 1: Docker 설치

```bash
# 프로젝트 디렉토리로 이동
cd /path/to/kim5257-gateway

# Docker 설치 스크립트 실행 (root 권한 필요)
sudo bash init_docker.sh
```

**설치 내용**
- Docker CE 최신 버전
- Docker Compose Plugin
- 현재 사용자를 docker 그룹에 추가

**설치 확인**
```bash
# 로그아웃 후 재로그인 필수!
exit

# 재로그인 후
docker --version
docker compose version
docker info
```

**예상 출력**
```
Docker version 24.x.x
Docker Compose version v2.x.x
```

### 3.3 Step 2: 데이터 디렉토리 생성

```bash
# 데이터 디렉토리 초기화 스크립트 실행
bash init_nfs.sh
```

> **참고**: 스크립트 이름은 `init_nfs.sh`이지만 NFS를 사용하지 않습니다. 단순히 로컬 디렉토리를 생성합니다.

**생성 확인**
```bash
ls -la /home/$USER/nfs/
```

### 3.4 Step 3: Certbot 설치

```bash
# Let's Encrypt Certbot 설치 (root 권한 필요)
sudo bash init_certbot.sh
```

**설치 확인**
```bash
certbot --version
```

**예상 출력**
```
certbot 2.x.x
```

---

## 4. 서비스 배포 절차

### 4.1 Gateway Stack 배포

#### 4.1.1 데이터 디렉토리 초기화

```bash
cd /path/to/kim5257-gateway

# Gateway 데이터 디렉토리 생성
bash init_data.sh
```

**생성되는 디렉토리**
```
/home/$USER/nfs/
├── kim5257-gateway-nginx/
│   ├── conf.d/
│   ├── htdocs/letsencrypt/
│   └── cert/
└── kim5257-gateway-nameserver/
    └── config/
```

#### 4.1.2 Gateway Stack 배포

```bash
# Docker Compose 파일 생성 및 배포
bash deploy.sh
```

**실행 내용**
1. `docker-compose.yml.template`에서 환경변수 치환
2. `docker-compose.yml` 생성
3. Docker Stack 배포

**배포 확인**
```bash
# Stack 확인
docker stack ls

# 서비스 확인
docker stack services kim5257_gateway

# 컨테이너 확인
docker ps | grep gateway
```

**예상 출력**
```
NAME                MODE         REPLICAS
kim5257_gateway     replicated   3/3
```

#### 4.1.3 도메인 및 TLS 인증서 설정

**신규 도메인 등록**
```bash
cd certbot

# 메인 도메인 등록 (Frontend)
bash new_cert.sh example.com msglog_web:3000

# API 서브도메인 등록 (Backend)
bash new_cert.sh api.example.com msglog_api:8080
```

**실행 프로세스**
1. Dummy 인증서로 Nginx 설정 생성
2. Nginx 설정 리로드
3. Let's Encrypt ACME Challenge 수행
4. 실제 인증서 발급 및 설치
5. Nginx 재리로드

**갱신 대상 도메인 등록**
```bash
# domain_list.txt에 도메인 추가
echo "example.com" >> certbot/domain_list.txt
echo "api.example.com" >> certbot/domain_list.txt
```

**자동 갱신 설정 (Crontab)**
```bash
crontab -e

# 매월 1일 오전 3시에 인증서 갱신
0 3 1 * * /home/$USER/workspace/kim5257-gateway/certbot/renew_cert.sh
```

**인증서 확인**
```bash
# 인증서 파일 확인
ls -la /home/$USER/nfs/kim5257-gateway-nginx/cert/example.com/

# Nginx 설정 확인
cat /home/$USER/nfs/kim5257-gateway-nginx/conf.d/example.com.conf

# HTTPS 접속 테스트
curl -I https://example.com
```

### 4.2 Database Stack 배포

#### 4.2.1 데이터 디렉토리 초기화

```bash
cd /path/to/kim5257-db

# Database 데이터 디렉토리 생성
bash init_data.sh
```

**생성되는 디렉토리**
```
/home/$USER/nfs/
└── kim5257-db-mariadb/
    └── data/
```

#### 4.2.2 환경변수 설정

```bash
# 환경변수 템플릿 복사
cp db.env.template db.env

# 환경변수 편집
nano db.env
```

**db.env 설정 예시**
```env
# MariaDB Root 비밀번호 (필수)
MYSQL_ROOT_PASSWORD=your_very_secure_root_password_here

# 애플리케이션 데이터베이스
MYSQL_DATABASE=msglog

# 애플리케이션 사용자
MYSQL_USER=msglog_user
MYSQL_PASSWORD=your_secure_app_password_here
```

> ⚠️ **보안 주의사항**
> - 강력한 비밀번호 사용 (최소 16자, 영문+숫자+특수문자)
> - `db.env` 파일은 절대 Git에 커밋하지 말것
> - 파일 권한 제한: `chmod 600 db.env`

#### 4.2.3 Database Stack 배포

```bash
# Stack 배포
bash deploy.sh
```

**배포 확인**
```bash
# Stack 확인
docker stack ls

# 서비스 확인
docker stack services kim5257_db

# 컨테이너 로그 확인
docker service logs -f kim5257_db_mariadb
```

**데이터베이스 접속 테스트**
```bash
# MariaDB 클라이언트 설치 (필요시)
sudo apt install mariadb-client

# 접속 테스트
mysql -h 127.0.0.1 -P 3306 -u msglog_user -p

# 비밀번호 입력 후
mysql> SHOW DATABASES;
mysql> USE msglog;
mysql> SHOW TABLES;
mysql> EXIT;
```

### 4.3 Backend 애플리케이션 배포

> **참고**: Backend 배포 절차는 별도 문서 참조

**일반적인 배포 흐름**
1. Backend 코드 서버에 업로드 (Git clone 또는 SFTP)
2. `.env` 파일 설정 (DB 연결 정보 등)
3. Docker 이미지 빌드
4. Docker Stack 배포

### 4.4 Frontend 애플리케이션 배포

Frontend는 **개발 환경에서 빌드 후 SFTP로 업로드**하는 방식을 사용합니다.

#### 4.4.1 로컬 환경에서 빌드

```bash
# 개발 PC에서 실행
cd frontend

# 의존성 설치
pnpm install

# 프로덕션 빌드
pnpm build
```

**빌드 결과**
- `dist/` 디렉토리에 정적 파일 생성
- `index.html`, JavaScript/CSS 번들
- 이미지 및 asset 파일

#### 4.4.2 서버로 파일 업로드

**방법 1: rsync 사용 (권장)**
```bash
# 개발 PC에서 실행
cd frontend

# rsync로 변경된 파일만 동기화
rsync -avz --delete \
  dist/ \
  user@server-ip:/home/user/nfs/kim5257-msg-log-prod-nginx/htdocs/

# 옵션 설명:
# -a: 아카이브 모드 (권한 유지)
# -v: 상세 출력
# -z: 압축 전송
# --delete: 불필요한 파일 삭제
```

**방법 2: SFTP 클라이언트 사용 (FileZilla, WinSCP 등)**
1. SFTP로 서버 접속
   - 호스트: 서버 IP
   - 포트: 22
   - 사용자: 서버 계정
2. 원격 디렉토리로 이동: `/home/$USER/nfs/kim5257-msg-log-prod-nginx/htdocs/`
3. 로컬 `dist/` 내용을 원격 `htdocs/`로 업로드

**방법 3: SCP + 압축**
```bash
# 개발 PC에서 실행
cd frontend

# 압축
tar -czf dist.tar.gz dist/

# 서버로 전송
scp dist.tar.gz user@server-ip:/tmp/

# 서버에서 압축 해제 및 배포
ssh user@server-ip << 'EOF'
cd /tmp
tar -xzf dist.tar.gz
rm -rf /home/$USER/nfs/kim5257-msg-log-prod-nginx/htdocs/*
cp -r dist/* /home/$USER/nfs/kim5257-msg-log-prod-nginx/htdocs/
rm -rf dist dist.tar.gz
EOF
```

#### 4.4.3 배포 확인

```bash
# 서버에서 실행
# 파일 존재 확인
ls -la /home/$USER/nfs/kim5257-msg-log-prod-nginx/htdocs/

# 필수 파일 체크
# - index.html
# - assets/ 디렉토리
# - favicon.ico

# 웹 브라우저에서 접속 테스트
curl https://example.com
```

---

## 5. 운영 및 유지보수

### 5.1 서비스 관리 명령어

#### 전체 Stack 관리

```bash
# 모든 Stack 목록
docker stack ls

# 모든 서비스 목록
docker service ls

# 특정 Stack의 서비스 목록
docker stack services kim5257_gateway
docker stack services kim5257_db
```

#### 개별 서비스 관리

```bash
# 서비스 상태 확인
docker service ps kim5257_gateway_nginx
docker service ps kim5257_db_mariadb

# 서비스 로그 확인
docker service logs -f kim5257_gateway_nginx
docker service logs -f kim5257_db_mariadb --tail 100

# 서비스 재시작
docker service update --force kim5257_gateway_nginx

# 서비스 스케일링
docker service scale kim5257_gateway_nginx=3
```

#### Stack 재배포

```bash
# Gateway Stack 재배포
cd /path/to/kim5257-gateway
bash deploy.sh

# Database Stack 재배포
cd /path/to/kim5257-db
bash deploy.sh
```

### 5.2 백업 및 복구

#### 데이터베이스 백업

```bash
# 백업 디렉토리 생성
mkdir -p ~/backups/mariadb

# 전체 데이터베이스 백업
docker exec $(docker ps -q -f name=mariadb) \
  mysqldump -u root -p'your_root_password' --all-databases \
  > ~/backups/mariadb/backup_$(date +%Y%m%d_%H%M%S).sql

# 특정 데이터베이스 백업
docker exec $(docker ps -q -f name=mariadb) \
  mysqldump -u root -p'your_root_password' msglog \
  > ~/backups/mariadb/msglog_$(date +%Y%m%d_%H%M%S).sql
```

#### 데이터베이스 복구

```bash
# 백업 파일 복구
docker exec -i $(docker ps -q -f name=mariadb) \
  mysql -u root -p'your_root_password' msglog \
  < ~/backups/mariadb/msglog_20240101_120000.sql
```

#### 자동 백업 스크립트 (Crontab)

```bash
crontab -e

# 매일 오전 2시에 백업
0 2 * * * /home/$USER/scripts/backup_mariadb.sh
```

**backup_mariadb.sh 예시**
```bash
#!/bin/bash
BACKUP_DIR="/home/$USER/backups/mariadb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# 백업 실행
docker exec $(docker ps -q -f name=mariadb) \
  mysqldump -u root -p'password' --all-databases \
  > "$BACKUP_DIR/backup_$TIMESTAMP.sql"

# 7일 이전 백업 삭제
find "$BACKUP_DIR" -name "backup_*.sql" -mtime +$RETENTION_DAYS -delete
```

### 5.3 모니터링

#### 시스템 리소스 확인

```bash
# 디스크 사용량
df -h

# 데이터 디렉토리 용량
du -sh /home/$USER/nfs/*

# 메모리 사용량
free -h

# CPU 사용량
top
htop  # 설치 필요: sudo apt install htop
```

#### Docker 리소스 확인

```bash
# 컨테이너별 리소스 사용량
docker stats

# 디스크 사용량
docker system df

# 불필요한 리소스 정리
docker system prune -a --volumes
```

#### 로그 관리

```bash
# 로그 크기 제한 설정 (daemon.json)
sudo nano /etc/docker/daemon.json
```

**daemon.json 예시**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
# Docker 재시작
sudo systemctl restart docker
```

### 5.4 보안 관리

#### 방화벽 설정 (UFW)

```bash
# UFW 활성화
sudo ufw enable

# 필수 포트 허용
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS

# 상태 확인
sudo ufw status verbose
```

#### SSL/TLS 인증서 갱신

```bash
# 수동 갱신
cd /path/to/kim5257-gateway/certbot
bash renew_cert.sh

# 갱신 로그 확인
tail -f /var/log/letsencrypt/letsencrypt.log
```

#### 패스워드 변경

**MariaDB 비밀번호 변경**
```bash
# MariaDB 컨테이너 접속
docker exec -it $(docker ps -q -f name=mariadb) mysql -u root -p

# 비밀번호 변경
ALTER USER 'msglog_user'@'%' IDENTIFIED BY 'new_password';
FLUSH PRIVILEGES;
EXIT;

# db.env 파일도 업데이트
nano /path/to/kim5257-db/db.env
```

### 5.5 정기 점검 체크리스트

#### 일간 점검
- [ ] 모든 서비스 정상 실행 확인
- [ ] 서비스 로그 에러 확인
- [ ] 디스크 용량 확인 (80% 이상 시 알림)

#### 주간 점검
- [ ] 백업 파일 정상 생성 확인
- [ ] 로그 파일 크기 확인
- [ ] 보안 업데이트 확인

#### 월간 점검
- [ ] TLS 인증서 갱신 확인
- [ ] 데이터베이스 백업 복구 테스트
- [ ] 시스템 보안 패치 적용
- [ ] 불필요한 Docker 이미지 정리

#### 분기별 점검
- [ ] Docker 이미지 업데이트
- [ ] 애플리케이션 버전 업데이트
- [ ] 용량 계획 검토
- [ ] 재해복구 시나리오 테스트

---

## 6. 트러블슈팅

### 6.1 Docker 관련 문제

#### 문제: Docker 명령어 실행 시 권한 오류

**증상**
```
permission denied while trying to connect to the Docker daemon socket
```

**원인**
- 현재 사용자가 docker 그룹에 속하지 않음
- 그룹 변경 후 로그아웃하지 않음

**해결방법**
```bash
# 사용자를 docker 그룹에 추가 (이미 실행했다면 생략)
sudo usermod -aG docker $USER

# 로그아웃 후 재로그인
exit

# 또는 임시로 그룹 적용
newgrp docker

# 확인
groups
docker ps
```

#### 문제: 컨테이너가 시작되지 않음

**증상**
```
replicated 0/1
```

**진단 및 해결**
```bash
# 서비스 상태 확인
docker service ps kim5257_gateway_nginx --no-trunc

# 에러 로그 확인
docker service logs kim5257_gateway_nginx

# 컨테이너 이벤트 확인
docker events --since 10m

# 일반적인 원인:
# 1. 포트 충돌 - 다른 프로세스가 포트 사용 중
sudo netstat -tlnp | grep :80
sudo netstat -tlnp | grep :443

# 2. 볼륨 마운트 실패 - 디렉토리 권한 문제
ls -la /home/$USER/nfs/

# 3. 이미지 Pull 실패 - 네트워크 또는 레지스트리 문제
docker pull nginx:latest
```

### 6.2 네트워크 관련 문제

#### 문제: 외부에서 웹사이트 접속 불가

**체크리스트**
```bash
# 1. Nginx 컨테이너 실행 확인
docker ps | grep nginx

# 2. 포트 바인딩 확인
sudo netstat -tlnp | grep :443

# 3. 방화벽 확인
sudo ufw status

# 4. DNS 확인
nslookup example.com
dig example.com

# 5. 로컬에서 접속 테스트
curl -I http://localhost
curl -I https://localhost -k

# 6. Nginx 설정 확인
docker exec $(docker ps -q -f name=nginx) nginx -t

# 7. Nginx 로그 확인
docker service logs kim5257_gateway_nginx
```

#### 문제: 컨테이너 간 통신 불가

**증상**
- Frontend에서 Backend API 호출 실패
- Backend에서 Database 연결 실패

**해결방법**
```bash
# 1. 네트워크 확인
docker network ls
docker network inspect kim5257_gateway_backbone

# 2. 서비스 이름으로 통신 테스트
docker exec $(docker ps -q -f name=nginx) ping -c 3 msglog_api
docker exec $(docker ps -q -f name=api) ping -c 3 mariadb

# 3. 네트워크 재생성 (최후의 수단)
docker network rm kim5257_gateway_backbone
docker network create --driver overlay kim5257_gateway_backbone
bash deploy.sh
```

### 6.3 TLS 인증서 관련 문제

#### 문제: Let's Encrypt 인증서 발급 실패

**증상**
```
Failed authorization procedure
Timeout during connect (likely firewall problem)
```

**원인 및 해결**
```bash
# 1. DNS 레코드 확인
nslookup example.com

# 2. 포트 80 외부 접근 확인
curl -I http://example.com/.well-known/acme-challenge/test

# 3. 방화벽 확인
sudo ufw status
sudo ufw allow 80/tcp

# 4. Nginx ACME Challenge 디렉토리 확인
ls -la /home/$USER/nfs/kim5257-gateway-nginx/htdocs/letsencrypt/

# 5. Certbot 수동 실행 (디버깅)
sudo certbot certonly --webroot \
  -w /home/$USER/nfs/kim5257-gateway-nginx/htdocs/letsencrypt \
  -d example.com \
  --dry-run

# 6. Rate Limit 확인
# Let's Encrypt는 주당 5번 실패 제한
# https://letsencrypt.org/docs/rate-limits/
```

#### 문제: HTTPS 접속 시 인증서 경고

**증상**
```
NET::ERR_CERT_AUTHORITY_INVALID
Your connection is not private
```

**원인 및 해결**
```bash
# 1. 인증서 파일 확인
ls -la /home/$USER/nfs/kim5257-gateway-nginx/cert/example.com/

# 2. 인증서 만료 확인
openssl x509 -in /home/$USER/nfs/kim5257-gateway-nginx/cert/example.com/fullchain.pem -noout -dates

# 3. Dummy 인증서 사용 중인지 확인
openssl x509 -in /home/$USER/nfs/kim5257-gateway-nginx/cert/example.com/fullchain.pem -noout -subject

# Dummy 인증서인 경우 "CN = Dummy Certificate" 출력됨

# 4. 실제 인증서로 교체
cd /path/to/kim5257-gateway/certbot
bash new_cert.sh example.com msglog_web:3000

# 5. Nginx 설정 확인
cat /home/$USER/nfs/kim5257-gateway-nginx/conf.d/example.com.conf

# 6. Nginx 재시작
docker service update --force kim5257_gateway_nginx
```

### 6.4 데이터베이스 관련 문제

#### 문제: MariaDB 접속 불가

**증상**
```
ERROR 2003 (HY000): Can't connect to MySQL server on 'localhost'
ERROR 1045 (28000): Access denied for user
```

**해결방법**
```bash
# 1. MariaDB 컨테이너 실행 확인
docker ps | grep mariadb

# 2. MariaDB 로그 확인
docker service logs kim5257_db_mariadb --tail 50

# 3. 포트 확인
sudo netstat -tlnp | grep 3306

# 4. 비밀번호 확인
cat /path/to/kim5257-db/db.env

# 5. 네트워크 연결 테스트
telnet localhost 3306

# 6. 컨테이너 내부에서 접속 테스트
docker exec -it $(docker ps -q -f name=mariadb) \
  mysql -u root -p

# 7. 사용자 권한 확인
docker exec -it $(docker ps -q -f name=mariadb) \
  mysql -u root -p -e "SELECT user, host FROM mysql.user;"

# 8. 서비스 재시작
docker service update --force kim5257_db_mariadb
```

#### 문제: 데이터베이스 성능 저하

**진단**
```bash
# 1. 슬로우 쿼리 로그 확인
docker exec $(docker ps -q -f name=mariadb) \
  tail -f /var/log/mysql/mysql-slow.log

# 2. 프로세스 리스트 확인
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p -e "SHOW PROCESSLIST;"

# 3. 테이블 상태 확인
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p msglog -e "SHOW TABLE STATUS;"

# 4. 인덱스 확인
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p msglog -e "SHOW INDEX FROM your_table;"
```

**최적화**
```bash
# 1. 테이블 최적화
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p msglog -e "OPTIMIZE TABLE your_table;"

# 2. 테이블 분석
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p msglog -e "ANALYZE TABLE your_table;"

# 3. 캐시 확인 및 설정
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p -e "SHOW VARIABLES LIKE 'query_cache%';"
```

### 6.5 디스크 관련 문제

#### 문제: 디스크 공간 부족

**증상**
```
No space left on device
```

**진단 및 해결**
```bash
# 1. 디스크 사용량 확인
df -h

# 2. 큰 파일/디렉토리 찾기
du -sh /home/$USER/nfs/* | sort -rh | head -10
du -sh /var/lib/docker/* | sort -rh | head -10

# 3. Docker 리소스 확인
docker system df

# 4. 불필요한 Docker 리소스 정리
docker system prune -a --volumes

# 확인 후 실행
docker image prune -a        # 사용하지 않는 이미지 삭제
docker volume prune          # 사용하지 않는 볼륨 삭제
docker container prune       # 중지된 컨테이너 삭제

# 5. 로그 파일 정리
sudo journalctl --vacuum-time=7d
sudo find /var/log -type f -name "*.log" -mtime +30 -delete

# 6. 오래된 백업 파일 삭제
find ~/backups -type f -mtime +30 -delete

# 7. MariaDB 바이너리 로그 정리
docker exec $(docker ps -q -f name=mariadb) \
  mysql -u root -p -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);"
```

### 6.6 Redis 관련 문제

#### 문제: Redis 연결 불가

**증상**
```
Could not connect to Redis
ECONNREFUSED
```

**해결방법**
```bash
# 1. Redis 컨테이너 확인
docker ps | grep redis

# 2. Redis 로그 확인
docker service logs kim5257_gateway_redis

# 3. Redis 연결 테스트
docker exec -it $(docker ps -q -f name=redis) redis-cli ping

# 예상 출력: PONG

# 4. Redis 정보 확인
docker exec -it $(docker ps -q -f name=redis) redis-cli INFO

# 5. 서비스 재시작
docker service update --force kim5257_gateway_redis
```

#### 문제: Redis 메모리 부족

**증상**
```
OOM command not allowed when used memory > 'maxmemory'
```

**해결방법**
```bash
# 1. 메모리 사용량 확인
docker exec $(docker ps -q -f name=redis) redis-cli INFO memory

# 2. 메모리 정책 확인
docker exec $(docker ps -q -f name=redis) redis-cli CONFIG GET maxmemory-policy

# 3. 불필요한 키 삭제
docker exec -it $(docker ps -q -f name=redis) redis-cli
> KEYS *
> DEL unnecessary_key

# 4. Redis 설정 변경 (필요시)
# docker-compose.yml에서 maxmemory 조정
```

### 6.7 Frontend 배포 관련 문제

#### 문제: 업로드한 파일이 반영되지 않음

**원인 및 해결**
```bash
# 1. 파일 업로드 확인
ls -la /home/$USER/nfs/kim5257-msg-log-prod-nginx/htdocs/

# 2. 파일 권한 확인
# 파일 소유자가 현재 사용자여야 함
sudo chown -R $USER:$USER /home/$USER/nfs/kim5257-msg-log-prod-nginx/htdocs/

# 3. Nginx 설정 확인
cat /home/$USER/nfs/kim5257-gateway-nginx/conf.d/example.com.conf

# 4. 브라우저 캐시 삭제
# Ctrl + Shift + R (강력 새로고침)
# 또는 개발자 도구에서 "Disable cache" 활성화

# 5. CDN/Proxy 캐시 문제 확인
curl -I https://example.com/assets/main.js

# 6. Nginx 캐시 헤더 확인
# Cache-Control, ETag 등
```

#### 문제: API 요청 CORS 에러

**증상**
```
Access to XMLHttpRequest has been blocked by CORS policy
```

**해결방법**
```bash
# 1. Backend API 서버의 CORS 설정 확인
# Backend 코드에서 도메인 허용 확인

# 2. Nginx에서 CORS 헤더 추가 (임시 해결)
# /home/$USER/nfs/kim5257-gateway-nginx/conf.d/example.com.conf

location /api {
    proxy_pass http://msglog_api:8080;
    
    # CORS 헤더 추가
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
    
    if ($request_method = 'OPTIONS') {
        return 204;
    }
}

# 3. Nginx 설정 리로드
docker exec $(docker ps -q -f name=nginx) nginx -s reload
```

### 6.8 일반적인 점검 절차

#### 전체 시스템 헬스체크

```bash
#!/bin/bash
# health_check.sh

echo "=== MSGLOG System Health Check ==="
echo ""

echo "1. Docker Service Status"
docker stack ls
echo ""

echo "2. Running Services"
docker service ls
echo ""

echo "3. Container Status"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
echo ""

echo "4. Disk Usage"
df -h | grep -E '(Filesystem|/dev/)'
echo ""

echo "5. Docker Disk Usage"
docker system df
echo ""

echo "6. Memory Usage"
free -h
echo ""

echo "7. Network Ports"
sudo netstat -tlnp | grep -E '(80|443|3306|6379)' | awk '{print $4, $7}'
echo ""

echo "8. Recent Errors (Last 10 lines)"
docker service logs kim5257_gateway_nginx --tail 10 2>&1 | grep -i error
docker service logs kim5257_db_mariadb --tail 10 2>&1 | grep -i error
echo ""

echo "=== Health Check Complete ==="
```

**실행**
```bash
chmod +x health_check.sh
./health_check.sh
```

---

## 7. 부록

### 7.1 유용한 명령어 모음

#### Docker 관리

```bash
# 모든 컨테이너 중지
docker stop $(docker ps -aq)

# 모든 컨테이너 삭제
docker rm $(docker ps -aq)

# 사용하지 않는 이미지 삭제
docker image prune -a

# 특정 컨테이너 로그 실시간 보기
docker logs -f <container_id>

# 컨테이너 리소스 사용량 실시간 모니터링
docker stats

# 컨테이너 내부 접속
docker exec -it <container_id> /bin/bash
docker exec -it <container_id> /bin/sh
```

#### 네트워크 진단

```bash
# 포트 스캔
nmap localhost

# 특정 포트 열림 확인
nc -zv localhost 80
nc -zv localhost 443

# DNS 조회
dig example.com
nslookup example.com

# 라우팅 추적
traceroute example.com

# 연결 테스트
curl -v https://example.com
wget --spider https://example.com
```

#### 시스템 모니터링

```bash
# CPU 사용량 Top 10
ps aux | sort -nrk 3,3 | head -n 10

# 메모리 사용량 Top 10
ps aux | sort -nrk 4,4 | head -n 10

# 디렉토리 크기 Top 10
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -n 10

# 실시간 로그 모니터링
tail -f /var/log/syslog
journalctl -f
```

### 7.2 환경변수 참조

#### Gateway Stack 환경변수

| 변수명 | 설명 | 기본값 |
|-------|------|--------|
| `USER` | 현재 사용자명 | 자동 설정 |
| `COMPOSE_PROJECT_NAME` | Docker Compose 프로젝트명 | `kim5257_gateway` |

#### Database Stack 환경변수

| 변수명 | 설명 | 필수 |
|-------|------|------|
| `MYSQL_ROOT_PASSWORD` | MariaDB root 비밀번호 | 필수 |
| `MYSQL_DATABASE` | 생성할 데이터베이스명 | 선택 |
| `MYSQL_USER` | 애플리케이션 사용자명 | 선택 |
| `MYSQL_PASSWORD` | 애플리케이션 사용자 비밀번호 | 선택 |

### 7.3 포트 매핑 참조

| 서비스 | 컨테이너 포트 | 호스트 포트 | 프로토콜 | 용도 |
|--------|-------------|-----------|---------|------|
| Nginx | 80 | 80 | HTTP | 웹 트래픽, ACME Challenge |
| Nginx | 443 | 443 | HTTPS | 보안 웹 트래픽 |
| MariaDB | 3306 | 3306 | TCP | 데이터베이스 연결 |
| Redis | 6379 | 6379 | TCP | 캐시/세션 스토어 |
| Nameserver | 53 | 53 | UDP/TCP | DNS 쿼리 |

### 7.4 파일 경로 참조

#### 호스트 시스템 경로

```
/home/$USER/
├── workspace/                          # 프로젝트 소스 코드
│   ├── kim5257-gateway/               # Gateway Stack
│   │   ├── docker-compose.yml
│   │   ├── init_docker.sh
│   │   ├── init_data.sh
│   │   ├── deploy.sh
│   │   └── certbot/
│   │       ├── new_cert.sh
│   │       ├── renew_cert.sh
│   │       └── domain_list.txt
│   │
│   └── kim5257-db/                    # Database Stack
│       ├── docker-compose.yml
│       ├── init_data.sh
│       ├── deploy.sh
│       ├── db.env.template
│       └── db.env
│
├── nfs/                               # 데이터 볼륨
│   ├── kim5257-gateway-nginx/
│   │   ├── conf.d/
│   │   ├── htdocs/letsencrypt/
│   │   └── cert/
│   │
│   ├── kim5257-msg-log-prod-nginx/
│   │   └── htdocs/
│   │
│   └── kim5257-db-mariadb/
│       └── data/
│
└── backups/                           # 백업 디렉토리
    └── mariadb/
```

#### 컨테이너 내부 경로

**Nginx 컨테이너**
```
/etc/nginx/conf.d/          # Nginx 도메인 설정
/usr/share/nginx/html/      # 웹 루트
/etc/letsencrypt/           # 인증서 디렉토리
```

**MariaDB 컨테이너**
```
/var/lib/mysql/             # 데이터 디렉토리
/var/log/mysql/             # 로그 디렉토리
/etc/mysql/                 # 설정 파일
```

**Redis 컨테이너**
```
/data/                      # 데이터 디렉토리
/etc/redis/                 # 설정 파일
```

### 7.5 주요 설정 파일 템플릿

#### Nginx 도메인 설정 예시

```nginx
# /home/$USER/nfs/kim5257-gateway-nginx/conf.d/example.com.conf

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    
    # ACME Challenge for Let's Encrypt
    location ^~ /.well-known/acme-challenge/ {
        root /usr/share/nginx/html/letsencrypt;
    }
    
    # Redirect to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL Certificate
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Proxy to Backend
    location / {
        proxy_pass http://msglog_web:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # API Proxy
    location /api/ {
        proxy_pass http://msglog_api:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # WebSocket Support (Socket.io)
    location /socket.io/ {
        proxy_pass http://msglog_api:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
