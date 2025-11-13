---
layout: page
title: "2025-11-12 - Git 히스토리에서 민감정보 제거 및 환경변수 관리"
date: 2025-11-12
categories: [til]
tags: [git, security, spring-boot, environment-variables]
---

# 2025-01-12 - Git 히스토리에서 민감정보 제거 및 환경변수 관리

## 📚 새로 배운 것

### 1. git filter-repo로 히스토리에서 파일 완전 제거

Git 히스토리에 AWS 자격 증명이 포함된 테스트 파일이 커밋되어 있어서 완전히 제거해야 했다.

```bash
# git filter-repo 사용 (git filter-branch보다 안전하고 빠름)
pip install git-filter-repo

# 특정 파일을 모든 히스토리에서 제거
git filter-repo --path src/test/java/kr/co/bdvr/webvms/infra/ipCam/IpCamServiceTest.java --invert-paths --force

# 주의: filter-repo는 원격 저장소 정보를 삭제함
git remote add origin https://github.com/user/repo.git

# 히스토리에서 민감정보가 제거되었는지 확인
git log --all --oneline -S "SENSITIVE_KEY"

# 강제 푸시
git push origin --force --all
```

**중요한 교훈:**

- `git filter-repo`는 원격 저장소(remote) 정보를 자동으로 삭제한다
- 작업 전 백업 브랜치를 반드시 만들어야 한다
- 팀원들은 저장소를 다시 클론해야 한다
- AWS 키가 노출되었다면 즉시 키를 교체해야 한다

### 2. Spring Boot에서 환경변수 관리 (spring-dotenv)

application.yml에 하드코딩된 민감정보를 .env 파일로 분리하고 git에 업로드 해서 관리한다.

**build.gradle 의존성 추가:**

```gradle
implementation 'me.paulschwarz:spring-dotenv:4.0.0'
```

**application.yml 환경변수 참조:**

```yaml
# 기존 (하드코딩)
datasource:
  username: dev
  password: pass

# 변경 후 (환경변수 참조)
datasource:
  username: ${DB_USERNAME:dev}  # 기본값 포함
  password: ${DB_PASSWORD}      # 기본값 없음 (필수)

# AWS 자격 증명
cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY}
      secret-key: ${AWS_SECRET_KEY}

# JWT 시크릿
jwt:
  secretKey: ${JWT_SECRET_KEY}
```

**.env 파일 구조:**

```bash
# Database Configuration (Production)
DB_HOST=192.168.0.14
DB_USERNAME=user_id
DB_PASSWORD=user_password

# AWS Configuration
AWS_ACCESS_KEY=your_key
AWS_SECRET_KEY=your_secret

# JWT Configuration
JWT_SECRET_KEY=your_secret_key
```

**.gitignore 수정:**

```gitignore
# 환경변수 파일 제외
.env

# 주의: application*.yml 패턴은 제거
# (이제 yml 파일은 깃으로 관리하고 민감정보는 .env로 분리)
```

**.env.example 생성:**
팀원들을 위한 템플릿 파일 제공

```bash
# .env.example
DB_USERNAME=user_id
DB_PASSWORD=user_password
AWS_ACCESS_KEY=aws_key
```

### 3. Git 커밋 취소 및 재작성

커밋을 취소하되 변경사항은 유지하고 싶을 때:

```bash
# 최근 1개 커밋 취소 (변경사항은 staged 상태로 유지)
git reset --soft HEAD~1

# 변경사항 확인
git status

# 추가 수정 후 재커밋
git add .
git commit -m "new commit message"
```

**옵션 차이:**

- `--soft`: 커밋만 취소, 변경사항은 staged 상태 유지
- `--mixed` (기본): 커밋 취소, 변경사항은 unstaged 상태
- `--hard`: 커밋과 변경사항 모두 삭제 (위험!)

## 🤔 궁금한 점

- git filter-repo vs BFG Repo-Cleaner의 성능 차이는?
- GitHub에 이미 푸시된 민감정보는 GitHub 측에서도 히스토리가 남을까?
- spring-dotenv vs Spring Cloud Config의 장단점은?
- 대규모 팀에서는 환경변수를 어떻게 관리하는게 좋을까? (AWS Secrets Manager, HashiCorp Vault 등)

## 📝 추가로 시도해볼 것

- [ ] GitHub Secret Scanning 알림 확인 및 설정
- [ ] AWS IAM 키 교체 자동화 스크립트 작성
- [ ] docker-compose.yml에서 env_file 사용해보기
- [ ] pre-commit hook으로 민감정보 커밋 방지
- [ ] application.yml validation - 필수 환경변수 검증 로직 추가
- [ ] 개발 환경과 운영 환경의 .env 파일 관리 프로세스 문서화

## 📌 참고 자료

- [git-filter-repo 공식 문서](https://github.com/newren/git-filter-repo)
- [spring-dotenv GitHub](https://github.com/paulschwarz/spring-dotenv)
- [AWS 자격 증명 모범 사례](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
