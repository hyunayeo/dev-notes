---
layout: page
title: "2025-08-18 - AI 분석 예외 처리 및 동시성 제어"
date: 2025-08-18
categories: [til]
tags: [java, ai, exception-handling, concurrency, spring]
permalink: /til/2025-08/2025-08-18-ai-analysis-exception-handling.md
---

# 2025-08-18 - AI 분석 예외 처리 및 동시성 제어

## 📚 새로 배운 것

### AI 분석 예외 처리 및 동시성 제어

#### 1. 커스텀 예외 처리 구현

- ErrorCode enum을 활용한 일관성 있는 예외 처리
- BusinessException + ErrorCode 방식으로 다국어 지원 구현

**추가된 에러 코드:**

```java
// ErrorCode.java
AI_ANALYSIS_REQUEST_FAILED(502, "AI_ANALYSIS_001", "error.ai.analysis.request.failed"),
AI_ANALYSIS_CREATION_FAILED(500, "AI_ANALYSIS_002", "error.ai.analysis.creation.failed"),
AI_SERVER_BUSY(429, "AI_ANALYSIS_003", "error.ai.server.busy"),
```

**다국어 메시지:**

- 한국어: "AI 서버와 통신 중입니다. 잠시 후 다시 시도해주세요."
- 영어: "AI server is busy. Please try again later."
- 일본어: "AI サーバーと通信中です。しばらくしてから再度お試しください。"

#### 2. AI 서버 동시성 제어

**문제 상황:**
AI 서버가 동시에 하나의 연결만 처리 가능한 제약이 있음

**이전 발생 오류 케이스:**

1. **동시성 오류 (Concurrency Issue)**
   - 여러 카메라에서 동시에 AI 분석 요청 시 AI 서버에서 처리 오류 발생
   - 데이터베이스 락(lock) 발생으로 인한 응답 지연 및 타임아웃
2. **AI 서버 응답 지연 시 DB 락 문제**
   - AI 서버 응답이 지연될 경우 트랜잭션이 길어져 데이터베이스 락 발생
   - 다른 요청들이 대기 상태에 빠지면서 전체 시스템 성능 저하
3. **Race Condition**
   - 동시에 여러 스레드가 `aiRequestInProgress` 체크 후 요청 시작
   - AI 서버에 중복 연결 시도로 인한 서버 불안정성

**해결 방법:**

```java
// AiAnalysisClient.java
private volatile boolean aiRequestInProgress = false; // volatile: 메모리 가시성 보장
private final Object aiRequestLock = new Object(); // 동시성 제어를 위한 락 객체 (race condition 방지)

private boolean tryStartAiRequest() {
    synchronized (aiRequestLock) {  // 한 번에 하나의 스레드만 진입
        if (aiRequestInProgress) {
            return false;  // 이미 진행 중이면 거부
        }
        aiRequestInProgress = true;  // 요청 시작 표시
        return true;
    }
}

private void finishAiRequest() {
    synchronized (aiRequestLock) {  // 안전하게 완료 표시
        aiRequestInProgress = false;
    }
}
```

**핵심 개념:**

- `volatile`: 메모리 가시성 보장 (다른 스레드에서 변경사항 즉시 확인)
- `synchronized`: 원자적 연산 보장 (체크-수정 과정을 하나의 단위로 처리)
- Race condition 방지를 위한 락 객체 사용

#### 3. 예외 처리 패턴 선택 기준

**BusinessException + ErrorCode 방식 사용:**

- 일반적인 비즈니스 로직 오류
- 단순한 메시지 전달 목적
- 새로운 기능의 예외
- 다국어 지원이 필요한 경우

**Custom Exception 클래스 사용:**

- 도메인별 핵심 예외 (NotFound, Duplicate 등)
- 특별한 필드나 메서드가 필요한 경우
- catch 블록에서 특별한 처리가 필요한 경우
- 타입 안전성이 중요한 경우

#### 4. 실제 적용 사례

**기존 코드:**

```java
// IllegalStateException 사용
return Mono.error(new IllegalStateException("AI 서버와 통신 중입니다. 잠시 후 다시 시도해주세요."));
```

**개선된 코드:**

```java
// BusinessException + ErrorCode 사용
return Mono.error(new BusinessException(ErrorCode.AI_SERVER_BUSY));
```

**장점:**

- HTTP 상태 코드 자동 매핑 (429 Too Many Requests)
- 다국어 메시지 자동 처리
- 일관성 있는 에러 응답 구조
- 중앙화된 에러 코드 관리

## 🤔 궁금한 점

- AI 서버의 동시 연결 제한이 하드웨어적 제약인지 소프트웨어적 제약인지?
- 더 효율적인 동시성 제어 방법이 있는지? (예: Semaphore, ReentrantLock)
- AI 서버 응답 시간이 길어질 경우의 타임아웃 처리 방안

## 📝 내일 해볼 것

- AI 서버 응답 타임아웃 설정 확인 및 최적화
- 다른 외부 API 통신에서도 동일한 패턴 적용 검토
- 에러 모니터링 및 로깅 개선 방안 검토
