# CorinEE V2 독립 운영 환경 구축 가이드

## 프로젝트 개요

### 목적
CorinEE v1과 v2를 하나의 공인 IP 서버에서 포트 분리를 통해 독립 운영하는 환경 구축

### 목표 아키텍처
- **v1 (corinee.site)**: 기존 포트 유지 (80, 443, 3000, 3306, 6379)
- **v2 (corinee-v2.site)**: 새로운 포트 할당 (8080, 8443, 3002, 3307, 6380)
- **완전 독립**: 별도 SSL 인증서, 데이터베이스, 컨테이너 환경

### 핵심 요구사항
- v1 서비스 중단 없이 v2 배포
- 각 버전별 독립적인 장애 격리
- 독립적인 배포 사이클 지원

---

## 배포 전략 설계

### 포트 분배 전략

| 서비스 | v1 포트 | v2 포트 | 목적 |
|--------|---------|---------|------|
| nginx HTTP | 80 | 8080 | 웹 서버 |
| nginx HTTPS | 443 | 8443 | 보안 웹 서버 |
| API Server | 3000 | 3002 | 백엔드 API |
| Frontend | 80, 443 | 3001, 3004 | 프론트엔드 |
| MySQL | 3306 | 3307 | 데이터베이스 |
| Redis | 6379 | 6380 | 캐시 서버 |

### 아키텍처 결정 과정

#### 고려된 옵션들
1. **Monolithic 방식**: 단일 docker-compose로 v1, v2 통합 관리
2. **Independent 방식**: 별도 docker-compose로 완전 분리 ✅ **선택**

#### 선택 이유
- **완전한 장애 격리**: v1/v2 서비스 간 독립성 보장
- **독립적 배포**: 각각의 배포 사이클 관리 가능
- **리소스 관리 유연성**: 개별 서비스별 리소스 할당

---

## SSL 인증서 설정

### v2 도메인 인증서 발급

#### 발급 과정
```bash
# 기존 nginx 서비스 일시 중지
sudo systemctl stop nginx

# v2 도메인 인증서 발급
sudo certbot certonly --standalone -d corinee-v2.site -d www.corinee-v2.site

# 기존 nginx 서비스 재시작
sudo systemctl start nginx
```

#### 🚨 트러블슈팅: 포트 80 충돌
**문제**: Let's Encrypt가 포트 80을 사용할 수 없어 인증서 발급 실패
```
Could not bind TCP port 80 because it is already in use by another process
```

**원인**: v1의 nginx가 포트 80을 점유하고 있어 certbot standalone 모드 실행 불가

**해결**: 기존 nginx를 잠시 중지하고 인증서 발급 후 재시작 (서비스 중단 시간: 약 1-3분)

#### 발급 결과 확인
```bash
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/corinee-v2.site/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/corinee-v2.site/privkey.pem
This certificate expires on 2025-09-28.
```

### 자동 갱신 설정

#### 자동 갱신 확인
```bash
# systemd timer 상태 확인
sudo systemctl status certbot.timer
sudo systemctl is-enabled certbot.timer

# 갱신 테스트 (실제 갱신 안함)
sudo certbot renew --dry-run
```

#### 갱신 스케줄
- **실행 주기**: 하루 2번 자동 체크
- **갱신 시점**: 만료 30일 전부터 시도
- **다음 실행**: 시스템에서 자동 관리

---

## Docker 컨테이너 설정

### docker-compose.yml 구성

#### 기본 서비스 구조
```yaml
version: "3.8"

services:
  server:     # 백엔드 API 서버
  client:     # 프론트엔드 (nginx 포함)
  mysql-v2:   # 데이터베이스
  redis-v2:   # 캐시 서버

networks:
  app-network:
    name: corinee_app-network_v2
    driver: bridge
```

#### 🚨 트러블슈팅: 포트 충돌 해결
**문제**: v1과 v2 컨테이너가 동일한 포트 사용으로 동시 실행 불가

**초기 설정 (문제)**:
```yaml
server:
  expose:
    - "3000"  # v1과 충돌
mysql-v2:
  expose:
    - "3306"  # v1과 충돌
```

**해결된 설정**:
```yaml
server:
  ports:
    - "3002:3000"  # 외부 3002 → 내부 3000
mysql-v2:
  ports:
    - "3307:3306"  # 외부 3307 → 내부 3306
```

### 네트워크 설정

#### v2 전용 네트워크
```yaml
networks:
  app-network:
    name: corinee_app-network_v2
    driver: bridge
```

#### 네트워크 분리 이점
- v1과 v2 컨테이너 간 네트워크 격리
- 내부 통신 보안 강화
- 네트워크 레벨 장애 격리

---

## nginx 설정

### nginx.conf 구성

#### 기본 구조
```nginx
events {
    worker_connections 1024;
}

http {
    # v2 도메인 서버 (HTTPS)
    server {
        listen 443 ssl;
        http2 on;
        server_name corinee-v2.site www.corinee-v2.site;
        
        # SSL 인증서 설정
        ssl_certificate /etc/letsencrypt/live/corinee-v2.site/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/corinee-v2.site/privkey.pem;
    }
    
    # HTTP to HTTPS 리다이렉트
    server {
        listen 80;
        server_name corinee-v2.site www.corinee-v2.site;
    }
}
```

#### 🚨 트러블슈팅: HTTP/2 설정 경고
**문제**: nginx 최신 버전에서 문법 변경으로 경고 발생
```
[warn] the "listen ... http2" directive is deprecated
```

**변경 전**:
```nginx
listen 443 ssl http2;
```

**변경 후**:
```nginx
listen 443 ssl;
http2 on;
```

#### 🚨 트러블슈팅: 리다이렉트 포트 불일치
**문제**: HTTP 접속 시 표준 443 포트로 리다이렉트되어 연결 실패

**변경 전**:
```nginx
return 301 https://$server_name$request_uri;  # 443 포트
```

**변경 후**:
```nginx
return 301 https://$server_name:8443$request_uri;  # 8443 포트
```

### SSL 보안 설정

#### 기본 보안 구성
```nginx
# SSL 프로토콜 설정
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
```

#### 추가 보안 헤더
```nginx
# 보안 헤더 (선택사항)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options DENY always;
add_header X-Content-Type-Options nosniff always;
```

---

## 컨테이너 배포

### 배포 순서

#### 1단계: 환경 준비
```bash
# 작업 디렉토리 생성
sudo mkdir -p /corinee-v2
cd /corinee-v2

# 필요 파일 업로드
# - docker-compose.yml
# - nginx.conf
# - .env
# - dockerfile-server-v2
# - dockerfile-client-v2
```

#### 2단계: 이미지 준비
```bash
# Docker Hub에서 이미지 pull
docker pull seunggwan/corinee-server-v2
docker pull seunggwan/corinee-client-v2

# 또는 로컬 빌드
docker-compose build
```

#### 3단계: 컨테이너 실행
```bash
# 전체 서비스 실행
docker-compose up -d

# 단계별 실행 (권장)
docker-compose up -d mysql-v2 redis-v2
sleep 10
docker-compose up -d server
sleep 5
docker-compose up -d client
```

### 🚨 트러블슈팅: 컨테이너 실행 실패

#### nginx 업스트림 호스트 오류
**문제**:
```
[emerg] host not found in upstream "corinee_client_v2" in /etc/nginx/nginx.conf:33
```

**원인**: 
1. v1과 v2 컨테이너가 혼재된 상태
2. nginx가 참조하는 컨테이너명과 실제 실행된 컨테이너명 불일치

**발견된 상황**:
```bash
# 실행 중인 컨테이너
corinee_client_1     # v1 (포트 80, 443 점유)
corinee_server_1     # v1 (포트 3000 점유)
corinee_nginx_v2     # v2
corinee_redis_v2     # v2
# v2 client/server는 포트 충돌로 실행 실패
```

**해결**: 컨테이너명 일치 및 포트 분리
```nginx
# nginx.conf 수정
proxy_pass http://client:80;    # 컨테이너 서비스명 사용
proxy_pass http://server:3000;
```

---

## 브라우저 SSL 인증

### SSL 인증서 검증

#### 🚨 트러블슈팅: 브라우저 SSL 경고
**문제**: 브라우저에서 "주의요함" 빨간색 경고 표시

**원인 분석**:
- 인증서 정보: `corinee.site` (v1용)
- 접속 도메인: `corinee-v2.site`
- **도메인 불일치로 인한 SSL 경고**

**해결 과정**:
1. nginx.conf에서 올바른 인증서 경로 확인
2. nginx 컨테이너 재시작으로 설정 적용
3. 브라우저에서 인증서 정보 재확인

#### SSL 등급 확인
**테스트 사이트**: SSL Labs (https://www.ssllabs.com/ssltest/)
- **입력**: `corinee-v2.site:8443`
- **예상 등급**: A 또는 A+

---

## 배포 검증

### 서비스 상태 확인

#### 컨테이너 상태
```bash
# 모든 v2 컨테이너 확인
docker-compose ps

# 실행 중인 컨테이너
docker ps | grep corinee

# 로그 확인
docker-compose logs nginx
docker-compose logs server
```

#### 네트워크 연결성
```bash
# 포트 바인딩 확인
sudo netstat -tlnp | grep -E "8080|8443|3002|3307|6380"

# SSL 인증서 확인
openssl s_client -connect corinee-v2.site:8443 -servername corinee-v2.site
```

### 기능 테스트

#### HTTP/HTTPS 접근성
```bash
# HTTP 접속 (리다이렉트 확인)
curl -I http://corinee-v2.site:8080

# HTTPS 접속
curl -I https://corinee-v2.site:8443

# API 엔드포인트 테스트
curl -I https://corinee-v2.site:8443/api/health
```

#### 데이터베이스 연결성
```bash
# MySQL 연결 테스트
mysql -h server-ip -P 3307 -u ${DB_USERNAME} -p

# Redis 연결 테스트
redis-cli -h server-ip -p 6380
```

---

## 운영 관리

### 모니터링

#### 로그 관리
```bash
# 실시간 로그 모니터링
docker-compose logs -f

# 특정 서비스 로그
docker-compose logs nginx
docker-compose logs server

# 로그 파일 위치
/var/log/nginx/access.log
/var/log/nginx/error.log
```

#### 리소스 모니터링
```bash
# 컨테이너 리소스 사용량
docker stats

# 디스크 사용량
docker system df
```

### 백업 및 복구

#### 데이터베이스 백업
```bash
# MySQL 백업
docker exec corinee_mysql_v2 mysqldump -u root -p${MYSQL_ROOT_PASSWORD_V2} ${MYSQL_DATABASE_V2} > backup.sql

# Redis 백업
docker exec corinee_redis_v2 redis-cli --rdb /data/dump.rdb
```

#### 설정 파일 백업
```bash
# 중요 설정 파일들
cp docker-compose.yml docker-compose.yml.backup
cp nginx.conf nginx.conf.backup
cp .env .env.backup
```

---

## 최종 접속 정보

### 사용자 접속 URL
- **v1 (기존)**: https://corinee.site
- **v2 (신규)**: https://corinee-v2.site:8443

### 관리자 접속 정보
- **v2 API 서버**: http://server-ip:3002
- **v2 MySQL**: server-ip:3307
- **v2 Redis**: server-ip:6380
- **v2 Frontend**: http://server-ip:3001

### 포트 요약
| 용도 | v1 포트 | v2 포트 | 상태 |
|------|---------|---------|------|
| Web (HTTP) | 80 | 8080 | ✅ 분리 완료 |
| Web (HTTPS) | 443 | 8443 | ✅ 분리 완료 |
| API Server | 3000 | 3002 | ✅ 분리 완료 |
| MySQL | 3306 | 3307 | ✅ 분리 완료 |
| Redis | 6379 | 6380 | ✅ 분리 완료 |

---

## 핵심 달성 사항

### ✅ 완료된 목표
- [x] v1과 v2 완전 독립 실행
- [x] 포트 충돌 해결 및 분리
- [x] SSL 인증서 정상 적용
- [x] 브라우저 보안 경고 해결
- [x] 자동 갱신 설정 완료
- [x] 리다이렉트 정상 동작

### 🛡️ 보안 및 안정성
- **SSL/TLS**: Let's Encrypt 인증서 적용
- **자동 갱신**: systemd timer를 통한 인증서 자동 갱신
- **네트워크 격리**: v1/v2 독립적인 네트워크 환경
- **장애 격리**: 개별 서비스 장애 시 상호 영향 없음

### 🚀 운영 효율성
- **독립 배포**: 각 버전별 독립적인 배포 사이클
- **리소스 분리**: 개별 서비스별 리소스 관리
- **확장성**: 추가 버전 배포 시 동일한 패턴 적용 가능 