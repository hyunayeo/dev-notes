---
layout: page
title: "2025-10-13 - Windows Server nginx Deployment"
date: 2025-10-13
categories: [til]
tags: [nginx, windows, deployment, spa, service]
---

# 2025-10-14 - Windows Server nginx Deployment

## 📚 새로 배운 것

### Windows에서 nginx로 SPA 배포하기

**1. nginx 설정 파일 구조**

- Windows에서는 절대 경로에 슬래시(`/`) 사용: `C:/path/to/file`
- 설정 파일 구조: `events { } → http { } → server { }`
- `server` 블록은 반드시 `http` 블록 안에 위치해야 함

**2. nginx location 우선순위**

```nginx
# 우선순위 순서:
location = /exact/path      # 1. 정확히 일치 (highest)
location ^~ /prefix/        # 2. prefix 일치 + 정규표현식 중단
location ~ /regex/          # 3. 정규표현식 (대소문자 구분)
location ~* /regex/         # 4. 정규표현식 (대소문자 무시)
location /prefix/           # 5. prefix 일치 (lowest)
```

**3. SPA 라우팅 설정**

```nginx
# 로그인 페이지 (루트)
location = / {
    try_files /index.html =404;
}

# 명시적 로그인 경로
location = /login {
    try_files /index.html =404;
}

# SPA 내부 라우트
location ~ ^/(live|log|channel|ai)$ {
    try_files /main.html =404;
}

# node_modules 매핑 (정규표현식보다 우선)
location ^~ /modules/ {
    alias C:/gitsource/aibox_front/node_modules/;
}
```

**4. 프론트엔드 config 엔드포인트 처리**

- `/api/config`는 프론트엔드가 환경 설정을 가져오는 엔드포인트
- nginx에서 직접 JSON 응답 제공:

```nginx
location = /api/config {
    default_type application/json;
    return 200 '{"apiBaseUrl":"http://localhost","wsBaseUrl":"http://localhost"}';
    add_header Content-Type application/json;
}
```

**5. API 프록시 설정**

```nginx
location /api/v1/ {
    proxy_pass https://localhost:8443/api/v1/;
    proxy_ssl_verify off;  # 자체 서명 인증서 사용 시
    # 기타 프록시 헤더들...
}
```

**6. Windows 서비스 등록 (NSSM)**

- PC 잠금/로그아웃 시에도 nginx가 종료되지 않도록 Windows 서비스로 등록
- NSSM (Non-Sucking Service Manager) 사용

설치 스크립트 주요 명령:

```powershell
# 서비스 설치
nssm install nginx C:\Dev\nginx\nginx.exe

# 작업 디렉토리 설정
nssm set nginx AppDirectory C:\Dev\nginx

# 자동 시작 설정
nssm set nginx Start SERVICE_AUTO_START

# 로그 설정
nssm set nginx AppStdout "C:\Dev\nginx\logs\service-stdout.log"
nssm set nginx AppStderr "C:\Dev\nginx\logs\service-stderr.log"
```

서비스 관리:

```powershell
net start nginx    # 시작
net stop nginx     # 중지
sc query nginx     # 상태 확인
```

**7. 일반적인 문제 해결**

| 문제                           | 원인                                         | 해결                                  |
| ------------------------------ | -------------------------------------------- | ------------------------------------- |
| "server" directive not allowed | `server` 블록이 `http` 밖에 있음             | `http { }` 안으로 이동                |
| 무한 리다이렉션 루프           | `location /`에서 계속 같은 파일로 리다이렉트 | location 순서 조정, `=` modifier 사용 |
| `/modules/` 404 에러           | 정규표현식 location이 먼저 매칭됨            | `^~` modifier로 우선순위 상승         |
| `undefined/api/v1/login`       | `API_BASE_URL`이 undefined                   | `/api/config` 엔드포인트 추가         |
| PC 잠금 시 nginx 종료          | 사용자 세션 프로세스로 실행                  | Windows 서비스로 등록                 |

## 🤔 궁금한 점

- NSSM 대신 Windows 기본 `sc.exe`로 서비스 등록할 수 있는지?
- nginx의 `proxy_pass`에서 trailing slash(`/`)가 있을 때와 없을 때의 정확한 차이
- Windows 환경에서 nginx worker process 최적화 방법
- SSL/TLS 인증서를 Let's Encrypt로 Windows에서 자동 갱신하는 방법

## 📝 추가로 시도해볼 것

- [ ] HTTPS 설정 추가 (WebRTC는 HTTPS 필수)
- [ ] nginx 로그 로테이션 자동화 설정
- [ ] 정적 파일 캐싱 효과 측정
- [ ] nginx 성능 모니터링 도구 설정
- [ ] 프로덕션 환경을 위한 보안 헤더 추가 검증
- [ ] 배포 자동화 스크립트 작성 (deploy.ps1)
- [ ] 백엔드 API 서버도 Windows 서비스로 등록
