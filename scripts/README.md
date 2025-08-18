# 문서화 명령어 가이드

Claude Code와의 대화 내용을 구조화된 문서로 자동 생성하는 커스텀 명령어들입니다.

## 📋 사용 가능한 명령어

### 1. `/doc-til` - 학습 내용 문서화
- **용도**: 새로 배운 기술, 개념, 방법론 정리
- **형식**: TIL (Today I Learned)
- **위치**: `dev-notes/til/YYYY-MM/`

```
/doc-til "AI 예외 처리" "Spring Boot"
```

### 2. `/doc-troubleshooting` - 문제 해결 과정 문서화
- **용도**: 버그 수정, 에러 해결, 트러블슈팅 과정 기록
- **형식**: Troubleshooting 문서
- **위치**: `dev-notes/troubleshooting/`

```
/doc-troubleshooting "동시성 오류 해결" "동시성"
```

### 3. `/doc-project` - 프로젝트 개발 기록 문서화
- **용도**: 기능 개발, 프로젝트 진행 과정 종합 기록
- **형식**: Project 개발 문서
- **위치**: `dev-notes/projects/`

```
/doc-project "WebVMS" "AI 분석 시스템"
```

## 🎯 언제 어떤 명령어를 사용할까?

### TIL 사용 시기
- ✅ 새로운 기술이나 라이브러리를 배웠을 때
- ✅ 개발 방법론이나 베스트 프랙티스를 익혔을 때
- ✅ 코드 패턴이나 설계 원칙을 학습했을 때
- ✅ 간단한 팁이나 트릭을 발견했을 때

### Troubleshooting 사용 시기
- 🐛 버그나 에러를 해결했을 때
- 🐛 성능 문제를 개선했을 때
- 🐛 설정이나 환경 문제를 해결했을 때
- 🐛 복잡한 디버깅 과정을 거쳤을 때

### Project 사용 시기
- 🚀 새로운 기능을 완전히 구현했을 때
- 🚀 시스템 아키텍처를 설계했을 때
- 🚀 여러 기술을 조합한 복합적인 작업을 했을 때
- 🚀 프로젝트 마일스톤을 달성했을 때

## 📝 사용 예시

### 오늘 대화의 경우
우리가 나눈 "AI 분석 예외 처리 및 동시성 제어" 대화는 다음과 같이 문서화할 수 있습니다:

1. **TIL 관점**: 새로 배운 동시성 제어 패턴
```
/doc-til "동시성 제어 패턴" "Java 멀티스레딩"
```

2. **Troubleshooting 관점**: AI 서버 동시성 문제 해결
```
/doc-troubleshooting "AI 서버 동시성 오류" "동시성"
```

3. **Project 관점**: WebVMS AI 분석 시스템 개선
```
/doc-project "WebVMS" "AI 분석 예외 처리 시스템"
```

## 🔄 명령어 실행 시 일어나는 일

1. **날짜 기반 파일명 생성**
2. **적절한 디렉토리에 파일 생성**
3. **해당 템플릿을 기반으로 구조화**
4. **대화 내용을 섹션별로 정리**
5. **코드 예시, 에러 메시지, 해결책 등 자동 포함**

## 💡 활용 팁

- **즉시 문서화**: 대화가 끝나자마자 바로 명령어 실행
- **적절한 제목**: 나중에 찾기 쉽도록 명확한 제목 사용
- **태그 활용**: 관련 기술스택이나 카테고리를 제목에 포함
- **정기적 리뷰**: 생성된 문서들을 주기적으로 검토하여 학습 효과 극대화

## 📁 파일 구조

```
dev-notes/
├── til/
│   ├── 2025-01/
│   │   ├── 2025-01-15-ai-exception-handling.md
│   │   └── 2025-01-15-concurrency-control.md
│   └── 2025-02/
├── troubleshooting/
│   ├── 2025-01-15-ai-server-concurrency-error.md
│   └── 2025-01-15-database-lock-issue.md
├── projects/
│   ├── WebVMS-AI-Analysis-2025-01-15.md
│   └── WebVMS-Exception-Handling-2025-01-15.md
└── scripts/
    ├── README.md
    ├── doc-til.md
    ├── doc-troubleshooting.md
    └── doc-project.md
```