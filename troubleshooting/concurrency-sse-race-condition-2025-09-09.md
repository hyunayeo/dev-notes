---
layout: page
title: "[Concurrency] SSE Emitter 관리에서 발생한 Race Condition"
date: 2025-09-09
categories: [troubleshooting, concurrency]
tags: [race-condition, sse, spring-boot, concurrency, thread-safety]
---

# [Concurrency] SSE Emitter 관리에서 발생한 Race Condition

**발생일:** 2025-09-09  
**프로젝트:** AiBox (AI 영상 분석 시스템)  
**카테고리:** 동시성 문제

## 🚨 문제 상황

### 증상

- SSE(Server-Sent Events) 연결이 간헐적으로 정상 종료되지 않음
- 메모리 누수로 의심되는 현상 (빈 Set 객체가 Map에 계속 남아있음)
- 다중 사용자 환경에서 이벤트 전송 실패가 불규칙적으로 발생

### 발생 환경

```java
// SseEventPushService.java - 문제가 있던 코드
private void registerEmitter(...) {
    Runnable cleanup = () -> {
        channelIds.forEach(channelId -> {
            Set<SseEmitter> emitters = emittersPerChannel.get(channelId);  // ← 1단계
            if (emitters == null) return;
            emitters.remove(emitter);                                     // ← 2단계
            if (emitters.isEmpty())                                       // ← 3단계 (race condition 발생 지점)
                emittersPerChannel.remove(channelId, emitters);           // ← 4단계
        });
    };
}
```

### 문제 재현 시나리오

1. 사용자 A가 SSE 연결 종료 (cleanup 실행 시작)
2. 동시에 사용자 B가 같은 채널에 SSE 연결 시작
3. 스레드 A: `emitters.isEmpty()` = true 확인
4. 스레드 B: 같은 Set에 새 emitter 추가
5. 스레드 A: 비어있지 않은 Set을 Map에서 제거 → 사용자 B의 연결 손실

## 🔍 원인 분석

### 근본 원인

**Non-atomic operation**: `isEmpty()` 체크와 `remove()` 연산 사이의 시간 간격에서 race condition 발생

### 세부 분석

1. **ConcurrentHashMap 사용의 한계**

   - Map 자체는 thread-safe하지만 복합 연산(여러 단계)은 원자적이지 않음
   - `get() → isEmpty() → remove()` 연산이 분리되어 있음

2. **타이밍 문제**

   ```
   Thread A (cleanup)           |  Thread B (register)
   ==========================  |  ==========================
   1. get() → [emitter1]       |
   2. remove(emitter1) → []    |
   3. isEmpty() = true         |  1. add(emitter2) → [emitter2]
                              |  2. Set is now non-empty
   4. remove() entire Set 💥   |  3. emitter2 lost!
   ```

3. **Double cleanup 문제**
   - `onTimeout()`과 `onError()` 콜백에서 중복 cleanup 실행
   - `emitter.complete()` 호출 시 `onCompletion()` 자동 실행되는데 추가로 `cleanup.run()` 호출

## 🛠 해결 과정

### 개선 방안

1. synchronized 블록 사용
   - 문제점 : 전체 Map에 대한 락으로 성능 저하 우려
2. ReentrantReadWriteLock
   - 문제점: 복잡성 증가, 오버엔지니어링
3. ConcurrentHashMap.compute() 사용

### 3번 적용: ConcurrentHashMap.compute() 사용

```java
// 수정된 코드
Runnable cleanup = () -> {
    channelIds.forEach(channelId -> {
        emitterMap.compute(channelId, (key, emitters) -> {
            if (emitters == null) return null;
            emitters.remove(emitter);
            return emitters.isEmpty() ? null : emitters;  // 원자적 연산
        });
    });
};
```

### ConcurrentHashMap의 원자적 연산 메서드들

- `compute()`: 키-값 쌍의 원자적 업데이트
- `computeIfPresent()`: 키가 존재할 때만 원자적 업데이트
- `computeIfAbsent()`: 키가 없을 때만 원자적 추가
- null 반환 시 해당 키를 Map에서 자동 제거

## 💡 배운 점

### 동시성 프로그래밍 원칙

1. **복합 연산의 원자성**: 여러 단계로 나뉜 연산은 원자적으로 처리해야 함
2. **ConcurrentHashMap의 활용**: `compute()`, `computeIfPresent()`, `computeIfAbsent()` 메서드 적극 활용
3. **Race Condition 탐지**: "만약 이 시점에 다른 스레드가 끼어든다면?" 질문하기

### 코드 리뷰 체크포인트

- [ ] 공유 데이터에 대한 복합 연산이 있는가?
- [ ] 원자적 연산으로 대체할 수 있는가?
- [ ] 콜백 함수에서 중복 실행 가능성은 없는가?

### 성능 vs 안전성

- `synchronized`: 안전하지만 성능 오버헤드
- `ConcurrentHashMap.compute()`: 안전하고 성능도 우수
- 적절한 도구 선택의 중요성

## 🔗 참고 자료

- [Java ConcurrentHashMap Documentation](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html)
- [Java Concurrency in Practice - Brian Goetz](https://www.oreilly.com/library/view/java-concurrency-in/0321349601/)
- [Spring Framework SSE Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-async-sse)
- [Race Condition 상세 분석](https://en.wikipedia.org/wiki/Race_condition)
- [ConcurrentHashMap vs synchronized performance comparison](https://stackoverflow.com/questions/1291836/concurrenthashmap-vs-synchronized-hashmap)
