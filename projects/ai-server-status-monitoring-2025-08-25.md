---
layout: page
title: "AI 서버 상태 모니터링 시스템"
date: 2025-08-25
categories: [projects]
tags: [Spring Boot, Java, 모니터링, 스케줄링, AI서버]
---

# AI 서버 상태 모니터링 시스템

## 📝 프로젝트 개요

- **기간**: 2025-01-25
- **기술스택**: Spring Boot 3.2.4, Java 17, Spring Scheduling, WebClient
- **역할**: 백엔드 개발
- **목적**: AI 서버의 상태를 주기적으로 모니터링하여 통신, 스트리밍, AI 분석, 클라우드 연결 상태를 개별적으로 추적하고 로그로 기록

## 🛠 주요 구현 사항

### 1. 시스템 로그 타입 확장

```java
// SystemLogType.java
AI_SERVER_COMMUNICATION(4001, "AI 서버 통신 상태"),
AI_SERVER_STREAMING(4002, "AI 서버 스트리밍 상태"),
AI_SERVER_ANALYSIS(4003, "AI 서버 분석 상태"),
AI_SERVER_CLOUD(4004, "AI 서버 클라우드 상태"),
```

### 2. 상태 모니터링 서비스

- **스케줄링**: `@Scheduled(fixedRate = 300000)` - 5분마다 실행
- **비동기 처리**: `AiAnalysisClient`의 WebClient를 활용한 비동기 API 호출
- **상태별 개별 관리**: 4가지 상태를 독립적으로 추적

### 3. 캐시 기반 상태 변경 감지

```java
// 통합 캐시 구조
private final Map<String, Boolean> statusCache = new ConcurrentHashMap<>();

// 키 구조
- "ai-server": 전역 통신 상태
- "streaming-{카메라ID}": 카메라별 스트리밍 상태
- "analysis-{카메라ID}": 카메라별 AI 분석 상태
- "cloud-{카메라ID}": 카메라별 클라우드 연결 상태
```

### 4. 상태 변경 감지 로직

- 이전 상태와 현재 상태 비교
- 변경된 상태만 선택적으로 로그 기록
- 최초 실행시 모든 상태를 로그에 기록

### 5. 로그 메시지 형식 표준화

```java
// 통신 상태: "[STATUS] on/off"
// 카메라별 상태: "[CAM_ID] {카메라ID} [STATUS] on/off"
```

### 6. AiAnalysisClient 확장

```java
public Mono<String> getAIServerStatus() {
    return sendRequestWithoutBody(HttpMethod.GET, "/ai-server/ai-analysis-channel");
}
```

## 🔗 참고 자료

### 기술 문서

- [Spring Scheduling Tasks](https://spring.io/guides/gs/scheduling-tasks/)
- [Spring WebClient Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-client)
