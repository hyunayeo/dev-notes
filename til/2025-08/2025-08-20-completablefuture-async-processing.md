---
layout: page
title: "2025-08-20 - CompletableFuture 비동기 처리"
date: 2025-08-20
categories: [til]
tags: [java, async, completablefuture, performance]
---

# 2025-08-20 - CompletableFuture 비동기 처리

## 📚 새로 배운 것

### BucketLogService 성능 최적화

Spring Boot 프로젝트에서 S3 버킷 로그 수집 서비스의 성능을 개선

**문제점:**

- 버킷별로 순차 처리 (N+1 성능 문제)
- 100개 버킷 × 5초 = 8분 20초 소요

**해결방법:**

```java
// 기존 순차 처리
buckets.forEach(bucket -> {
    collectBucketUsage(bucket, month);
});

// 개선된 비동기 처리
List<CompletableFuture<Void>> bucketFutures = buckets.stream()
    .map(bucket -> CompletableFuture.runAsync(() -> collectBucketUsage(bucket, month), executorService))
    .toList();

CompletableFuture<Void> allBucketTasks = CompletableFuture.allOf(
    bucketFutures.toArray(new CompletableFuture[0])
);

allBucketTasks.join(); // 모든 작업 완료 대기
```

### CompletableFuture 핵심 개념

1. **runAsync()**: 반환값 없는 비동기 작업
2. **allOf()**: 여러 CompletableFuture를 하나로 결합
3. **join()**: 작업 완료까지 블로킹 대기 (RuntimeException 발생)
4. **get()**: 작업 완료까지 블로킹 대기 (검사 예외 발생)

### 리소스 관리

```java
private final ExecutorService executorService = Executors.newFixedThreadPool(10);

@PreDestroy
public void cleanup() {
    if (executorService != null && !executorService.isShutdown()) {
        executorService.shutdown();
        log.info("ExecutorService 종료 완료");
    }
}
```

**결과:** 약 10배 성능 향상 (100개 버킷: 8분 → 50초)

## 🤔 궁금한 점

- 스레드풀 크기 최적화 방법 (현재 10개로 설정)
- AWS API 호출 시 rate limiting 고려사항
- CompletableFuture 예외 처리 모범 사례
- 대량 데이터 처리 시 메모리 사용량 최적화

## 📝 내일 해볼 것

- AWS credential 검증 및 에러 핸들링 추가
- Circuit breaker 패턴 적용
- UTC 타임존 일관성 확보
- 입력 검증 로직 강화
- 모니터링 및 알림 기능 추가
