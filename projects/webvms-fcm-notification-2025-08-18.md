---
layout: page
title: "WebVMS FCM 알림 시스템 개선"
date: 2025-08-18
categories: [projects]
tags: [webvms, fcm, push-notification, spring-boot, performance]
---

# WebVMS FCM 알림 시스템 개선

## 📝 프로젝트 개요

- **기간**: 2025-08-18
- **기술스택**: Spring Boot, JPA/QueryDSL, Firebase Cloud Messaging, MariaDB
- **역할**: Backend Developer
- **목적**: 사용자별 알림 on/off 기능 구현 및 FCM 발송 로직 최적화

## 🛠 주요 구현 사항

### 1. 기존 FCM 발송 버그 수정

- **문제**: `users.get(0)`으로 첫 번째 사용자만 처리하던 버그
- **해결**: 모든 사용자의 토큰을 수집하도록 `flatMap` 사용

```java
// Before (버그)
List<UserToken> userTokens = userTokenRepository.findAllByUser(users.get(0));

// After (수정)
List<String> tokens = users.stream()
    .flatMap(user -> userTokenRepository.findAllByUser(user).stream())
    .map(UserToken::getPushToken)
    .filter(token -> token != null && !token.isBlank())
    .toList();
```

### 2. 사용자별 알림 on/off 기능 구현

#### 설정 체계

- `NOTIFY_ALL`: 전체 알림 on/off (boolean)
- `NOTIFY_PER_CAMERA`: 카메라별 알림 설정 (JSON)

#### 구현된 메서드

```java
private boolean shouldNotifyUser(User user, Camera camera) {
    // 1. 전체 알림 설정 확인 (기본값: true)
    boolean notifyAll = userSettingRepository.findByUserAndSettingCode(user, UserSettingCode.NOTIFY_ALL)
        .map(setting -> Boolean.parseBoolean(setting.getValue()))
        .orElse(true);

    if (!notifyAll) {
        return false;
    }

    // 2. 카메라별 알림 설정 확인
    return userSettingRepository.findByUserAndSettingCode(user, UserSettingCode.NOTIFY_PER_CAMERA)
        .map(setting -> {
            try {
                @SuppressWarnings("unchecked")
                Map<String, Boolean> cameraSettings = JsonUtils.fromJson(setting.getValue(), Map.class);
                return cameraSettings.getOrDefault(camera.getCamId().toString(), true);
            } catch (Exception e) {
                log.warn("Failed to parse camera notification settings for user: {}, using default", user.getUserId(), e);
                return true;
            }
        })
        .orElse(true);
}
```

### 3. 쿼리 성능 최적화

#### N+1 쿼리 문제 해결

**UserTokenRepository**에 일괄 조회 메서드 추가:

```java
List<UserToken> findAllByUsers(List<User> users);
```

#### 성능 개선 결과

- **Before**: 사용자 수(N) + 1번의 쿼리 실행
- **After**: 1번의 쿼리로 모든 토큰 조회

### 4. FCM 발송 트랜잭션 분리

#### 문제점

- FCM 발송이 트랜잭션 내부에서 실행
- 외부 API 호출로 인한 트랜잭션 장시간 유지
- FCM 실패 시 전체 이벤트 처리 롤백 위험

#### 해결책

```java
// 트랜잭션 분리를 위한 새로운 @Async 메서드
@Async
@Transactional(readOnly = true)
public void sendPushAlarmAsync(Long eventId) {
    AiEvent aiEvent = eventRepository.findById(eventId)
        .orElseThrow(() -> new EventNotFoundException());
    sendPushAlarm(aiEvent);
}

// createEvent에서 호출 방식 변경
// Before: sendPushAlarm(aiEvent);
// After: sendPushAlarmAsync(aiEvent.getEventId());
```

### 배운 점

- 트랜잭션 경계를 고려한 설계의 중요성
- 외부 API 호출과 DB 트랜잭션 분리의 필요성

## 🔗 참고 자료

- [Spring Boot @Async 사용법](https://spring.io/guides/gs/async-method/)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [JPA N+1 Query Problem 해결법](https://jojoldu.tistory.com/165)
