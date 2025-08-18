# /doc-troubleshooting 명령어

대화 내용을 Troubleshooting 형식으로 문서화합니다.

## 사용법

```
/doc-troubleshooting [문제제목] [카테고리]
```

### 예시
```
/doc-troubleshooting "AI 서버 동시성 오류" "동시성"
```

## 실행되는 작업

1. **파일명 생성**: `YYYY-MM-DD-{문제제목}.md`
2. **경로**: `C:\gitsource\dev-notes\troubleshooting\`
3. **템플릿 적용**: Troubleshooting 템플릿 기반으로 구조화
4. **내용 정리**: 문제 상황, 원인, 해결 과정을 체계적으로 정리

## 생성되는 문서 구조

- 🚨 문제 상황
- 🔍 원인 분석
- 🛠 해결 과정
- 💡 배운 점
- 🔗 참고 자료

## 활용 팁

- 에러나 버그를 해결한 경우 사용
- 문제 상황과 해결 과정을 상세히 기록
- 동일한 문제 재발 시 빠른 해결을 위한 레퍼런스