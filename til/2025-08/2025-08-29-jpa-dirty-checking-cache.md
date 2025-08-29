---
layout: page
title: "2025-08-29 - JPA Dirty Checking과 캐시 동기화 이슈"
date: 2025-08-29
categories: [til]
tags: [JPA, Spring, Cache, Dirty Checking]
---

# 2025-08-29 - JPA Dirty Checking과 캐시 동기화 이슈

## 📚 새로 배운 것

### JPA Dirty Checking의 동작 조건

- `@Transactional` 메서드 내에서 **영속성 컨텍스트에 관리되는 엔티티**를 수정할 때만 자동 저장
- 실제 엔티티 변경사항이 있어야 UPDATE 쿼리 실행
- 변경사항이 없으면 트랜잭션 커밋해도 DB 업데이트 발생하지 않음

### 자동 저장되지 않는 경우들

1. **@Transactional이 없는 경우**
2. **준영속(Detached) 상태의 엔티티**
3. **새로운 엔티티 객체**
4. **트랜잭션이 rollback되는 경우**

```java
// 저장 안됨 - @Transactional 없음
public void updateUser(Long id) {
    User user = userRepository.findById(id).get();
    user.setName("New Name"); // 변경사항이 DB에 반영되지 않음
}

// 저장 안됨 - 준영속 상태
@Transactional
public void updateUser(Long id) {
    User user = userRepository.findById(id).get();
    entityManager.detach(user); // 영속성 컨텍스트에서 분리
    user.setName("New Name"); // 저장 안됨
}
```

### 캐시와 저장의 타이밍 이슈

Spring Cache(`@CacheEvict`)와 JPA Dirty Checking 사용 시 주의사항:

1. **캐시 무효화 타이밍**: `@CacheEvict`는 메서드 실행 직후 캐시 삭제
2. **DB 저장 타이밍**: Dirty Checking은 트랜잭션 커밋 시점에 DB 저장
3. **타이밍 차이**: 캐시는 삭제했는데 DB 변경은 아직 커밋 전일 수 있음

```java
@CacheEvict(value = "userByUid", key = "#user.userUid")
public UserResponse updateMyInfo(User user, UpdateMyInfoRequest request) {
    // 변경사항이 없는 경우
    if (조건문_모두_false) {
        // 캐시는 삭제되었지만 DB는 변경되지 않음
        // 다음 조회 시 변경되지 않은 데이터가 다시 캐시됨
    }
}
```

해결책

```java

  @CacheEvict(value = "userByUid", key = "#user.userUid")
  public UserResponse updateMyInfo(User user,
  UpdateMyInfoRequest request) {
      // ... 로직 ...

      // 명시적 save() 호출로 즉시 DB 반영 보장
      userRepository.save(user);
      return new UserResponse(user);
  }

```

캐시 사용 시에는 save() 호출을 유지하는 것이 안전함.

- save() 호출 → 즉시 DB에 변경사항 반영
- 캐시 무효화와 DB 변경 사이의 타이밍 갭 최소화
- 일관성 보장

### 더 나은 해결책들

1. 조건부 캐시 무효화

```java
@CacheEvict(value = "userByUid", key = "#user.userUid", condition = "#result != null")
```

2. 트랜잭션 커밋 후 캐시 무효화

```java
@TransactionalEventListener(phase =  TransactionPhase.AFTER_COMMIT)
```

## 🤔 궁금한 점

- `@CacheEvict`의 실행 시점을 트랜잭션 커밋 후로 변경할 수 있는 방법이 있을까?
- 대용량 데이터 처리 시 명시적 `save()` vs Dirty Checking의 성능 차이는 어느 정도일까?
- 캐시 무효화 전략에서 조건부 무효화 방법은?

## 📝 내일 해볼 것

- Spring Cache의 다양한 무효화 전략 테스트
- JPA 배치 업데이트와 캐시 동기화 관련 Best Practice 찾아보기
- 현재 프로젝트의 다른 캐시 사용 부분도 동일한 이슈가 있는지 확인
