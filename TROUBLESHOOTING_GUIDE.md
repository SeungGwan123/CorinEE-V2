# CorinEE V2 독립 운영 환경 구축 - 트러블슈팅 가이드

## 프로젝트 개요
CorinEE v1과 v2를 하나의 서버에서 포트 분리를 통해 독립 운영하는 환경 구축 프로젝트의 트러블슈팅 과정을 상세히 기록합니다.

## 목표 아키텍처
- **v1**: 기존 포트 유지 (80, 443, 3000, 3306, 6379)
- **v2**: 새로운 포트 사용 (8080, 8443, 3002, 3307, 6380)
- **완전 독립**: 서로 다른 SSL 인증서, 데이터베이스, 컨테이너 환경

---

## 🚨 주요 트러블슈팅 이슈들

### 1. SSL 인증서 발급 실패 - 포트 80 충돌

#### **문제 상황:**
```
Could not bind TCP port 80 because it is already in use by another process on
this system (such as a web server). Please stop the program in question and then
try again.
```

#### **원인 분석:**
- v1의 nginx가 포트 80을 점유하고 있어 Let's Encrypt certbot이 standalone 모드로 인증서 발급 불가
- certbot standalone 모드는 HTTP-01 챌린지를 위해 포트 80 필요

#### **해결 방법:**
기존 nginx 서비스를 잠시 중지하고 인증서 발급 후 재시작:
```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d corinee-v2.site -d www.corinee-v2.site
sudo systemctl start nginx
```

#### **결과:**
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/corinee-v2.site/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/corinee-v2.site/privkey.pem
This certificate expires on 2025-09-28.
```

---

### 2. HTTPS 리다이렉트 포트 불일치

#### **문제 상황:**
- HTTP 접속 시 표준 443 포트로 리다이렉트되어 연결 실패
- v2는 8443 포트를 사용하므로 리다이렉트 URL이 잘못됨

#### **원인 분석:**
nginx.conf의 리다이렉트 설정이 표준 포트로 하드코딩되어 있음:
```nginx
location / {
    return 301 https://$server_name$request_uri;  # 443 포트로 리다이렉트
}
```

#### **해결 방법:**
nginx.conf 수정하여 8443 포트로 리다이렉트하도록 변경:
```nginx
location / {
    return 301 https://$server_name:8443$request_uri;  # 8443 포트로 리다이렉트
}
```

---

### 3. 브라우저 SSL 경고 - 잘못된 인증서 사용

#### **문제 상황:**
- 브라우저에서 "주의요함" 빨간색 경고 표시
- 인증서 정보 확인 결과 `corinee.site` 인증서가 사용됨 (v1용)
- 도메인 불일치: 접속(`corinee-v2.site`) vs 인증서(`corinee.site`)

#### **원인 분석:**
nginx 컨테이너가 올바른 인증서 경로를 로드하지 못함

#### **해결 방법:**
1. nginx.conf에서 올바른 인증서 경로 확인:
```nginx
ssl_certificate /etc/letsencrypt/live/corinee-v2.site/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/corinee-v2.site/privkey.pem;
```

2. nginx 컨테이너 재시작으로 설정 적용:
```bash
docker-compose restart nginx
```

---

### 4. nginx 컨테이너 실행 실패 - 업스트림 호스트 없음

#### **문제 상황:**
```
[emerg] host not found in upstream "corinee_client_v2" in /etc/nginx/nginx.conf:33
```

#### **원인 분석:**
1. **컨테이너 이름 불일치**: nginx가 찾는 컨테이너명과 실제 실행된 컨테이너명이 다름
2. **의존성 순서 문제**: nginx가 client/server 컨테이너보다 먼저 실행되어 네트워크 연결 실패

#### **발견된 실행 상태:**
```bash
# v1과 v2가 혼재된 상태
corinee_client_1     # v1 컨테이너 (포트 80, 443 점유)
corinee_server_1     # v1 컨테이너 (포트 3000 점유)
corinee_nginx_v2     # v2 컨테이너
corinee_redis_v2     # v2 컨테이너
# v2 client/server 컨테이너는 포트 충돌로 실행 실패
```

---

### 5. HTTP/2 설정 경고

#### **문제 상황:**
```
[warn] the "listen ... http2" directive is deprecated, use the "http2" directive instead
```

#### **원인 분석:**
nginx 최신 버전에서 HTTP/2 설정 문법 변경

#### **해결 방법:**
nginx.conf 문법 업데이트:
```nginx
# 변경 전
listen 443 ssl http2;

# 변경 후  
listen 443 ssl;
http2 on;
```

---

### 6. 포트 충돌 해결 - 완전 분리 아키텍처

#### **최종 문제:**
v1과 v2가 동일한 포트들을 사용하여 동시 운영 불가

#### **해결 전략:**
모든 v2 서비스 포트를 v1과 다르게 분리

#### **docker-compose.yml 수정:**

**변경 전 (expose only):**
```yaml
server:
  expose:
    - "3000"
client:
  expose:
    - "80"
    - "443"
mysql-v2:
  expose:
    - "3306"
redis-v2:
  expose:
    - "6379"
```

**변경 후 (ports mapping):**
```yaml
server:
  ports:
    - "3002:3000"
client:
  ports:
    - "3001:80"
    - "3004:443"
mysql-v2:
  ports:
    - "3307:3306"
redis-v2:
  ports:
    - "6380:6379"
```

#### **nginx.conf 컨테이너명 수정:**
```nginx
# 변경 전
proxy_pass http://corinee_client_v2:80;
proxy_pass http://corinee_server_v2:3000;

# 변경 후
proxy_pass http://client:80;
proxy_pass http://server:3000;
```

---

## 📊 최종 포트 분배

| 서비스 | v1 포트 | v2 포트 | 비고 |
|--------|---------|---------|------|
| nginx HTTP | 80 | 8080 | 외부 접근 |
| nginx HTTPS | 443 | 8443 | 외부 접근 |
| server | 3000 | 3002 | API 서버 |
| client | 80, 443 | 3001, 3004 | 내부 통신 |
| mysql | 3306 | 3307 | 데이터베이스 |
| redis | 6379 | 6380 | 캐시 서버 |

---

## 🔧 주요 설정 파일 변경사항

### 1. nginx.conf
```nginx
# HTTP/2 설정 업데이트
listen 443 ssl;
http2 on;

# 포트별 리다이렉트
return 301 https://$server_name:8443$request_uri;

# 컨테이너명 단순화
proxy_pass http://client:80;
proxy_pass http://server:3000;

# v2 전용 SSL 인증서
ssl_certificate /etc/letsencrypt/live/corinee-v2.site/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/corinee-v2.site/privkey.pem;
```

### 2. docker-compose.yml
```yaml
# 모든 서비스를 expose에서 ports로 변경
# 포트 충돌 방지를 위한 완전 분리
# v2 전용 네이밍 (mysql-v2, redis-v2)
```

---

## 🚀 배포 검증 단계

### 1. SSL 인증서 확인
```bash
sudo certbot certificates
openssl s_client -connect corinee-v2.site:8443 -servername corinee-v2.site
```

### 2. 컨테이너 상태 확인  
```bash
docker-compose ps
docker ps | grep corinee
```

### 3. 서비스 접근성 테스트
```bash
curl -I http://corinee-v2.site:8080
curl -I https://corinee-v2.site:8443
```

### 4. 자동 갱신 확인
```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

---

## 💡 교훈 및 개선점

### 1. 포트 설계의 중요성
- 초기부터 포트 분리를 계획했다면 트러블슈팅 시간 단축 가능
- expose vs ports 차이에 대한 명확한 이해 필요

### 2. SSL 인증서 관리
- 도메인별 독립적인 인증서 관리 필요
- Let's Encrypt 자동 갱신 검증 중요

### 3. 컨테이너 의존성 관리
- nginx가 참조하는 컨테이너들의 실행 순서 고려
- 네트워크 설정과 컨테이너명 일치성 확인

### 4. 설정 버전 관리
- nginx 설정 문법 변경 사항 추적
- Docker Compose 스키마 업데이트 대응

---

## 🔍 최종 확인 사항

- [x] v1과 v2 독립 실행 확인
- [x] SSL 인증서 정상 적용
- [x] 포트 충돌 해결
- [x] 자동 갱신 설정 완료
- [x] 브라우저 보안 경고 해결
- [x] 리다이렉트 정상 동작

---

## 📞 접속 정보

### 사용자 접속
- **v1**: https://corinee.site
- **v2**: https://corinee-v2.site:8443

### 관리자 접속
- **v2 API**: http://server-ip:3002
- **v2 DB**: server-ip:3307
- **v2 Redis**: server-ip:6380 