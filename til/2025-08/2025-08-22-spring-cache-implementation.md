---
layout: page
title: "2025-08-22 - Spring Boot Cache 구현"
date: 2025-08-22
categories: [til]
tags: [spring, cache, caffeine, performance, optimization]
---

# 2025-08-22 - Spring Boot Cache 구현

## 📚 새로 배운 것

### Spring Cache 기본 구성

Spring Boot에서 캐시를 구현하기 위한 기본 설정:

```java
@Configuration
@EnableCaching  // 캐시 기능 활성화
public class CommonConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)           // 최대 1000개 엔트리
            .expireAfterWrite(Duration.ofMinutes(30))  // 30분 후 만료
            .recordStats());             // 통계 수집 활성화
        return cacheManager;
    }
}
```

### 캐시 어노테이션 활용

- `@Cacheable`: 결과를 캐시에 저장하고 조회
- `@CacheEvict`: 캐시 무효화 (데이터 변경 시)
- `@CachePut`: 캐시 갱신

```java
@Cacheable(value = "userByUid", key = "#id")
public User findByUserUid(String id) {
    return userRepository.findByUserUid(id)
        .orElseThrow(() -> new IllegalArgumentException("Invalid user uid"));
}

@CacheEvict(value = {"currentUser", "userByUid"}, key = "#user.userUid")
public void incrementInvalidPasswordCount(User user) {
    user.incrementInvalidPasswdCnt();
    userRepository.save(user);
}
```

### 캐시 키 전략

SpEL(Spring Expression Language)을 사용한 캐시 키 설정:

- `key = "#principal.name"`: 메서드 파라미터의 속성 접근
- `key = "#id"`: 메서드 파라미터 직접 사용
- `key = "#user.userUid"`: 객체의 속성 접근

### 성능 개선 효과

실제 측정된 성능 향상:

- 첫 번째 요청: DB 조회 (30ms)
- 두 번째 요청: 캐시 조회 (1ms 미만)
- 평균 응답시간 70-80% 단축

### 테스트 코드 작성

캐시 동작을 검증하는 테스트:

```java
@Test
@DisplayName("findByUserUid 메서드 캐시 동작 테스트")
void testFindByUserUidCache() {
    // 첫 번째 호출
    User result1 = userService.findByUserUid(userUid);
    // 두 번째 호출
    User result2 = userService.findByUserUid(userUid);

    // DB 조회는 1번만 발생해야 함
    verify(userRepository, times(1)).findByUserUid(userUid);

    // 캐시에 데이터가 저장되었는지 확인
    assertThat(cacheManager.getCache("userByUid").get(userUid)).isNotNull();
}
```

## 🤔 궁금한 점

- 분산 환경에서 캐시 동기화는 어떻게 처리할까? (Redis 등)
- 캐시 메모리 사용량을 실시간으로 모니터링하는 방법은?
- 캐시 hit/miss 비율을 측정하고 최적화하는 전략은?
- 대용량 데이터에서 캐시 키 충돌을 방지하는 방법은?

## 📝 내일 해볼 것

- [ ] 다른 자주 조회되는 데이터(사이트 정보, 대시보드 통계)에 캐시 적용
- [ ] 캐시 통계 모니터링 대시보드 구현
- [ ] Redis를 사용한 분산 캐시 구현 고려
- [ ] 캐시 성능 벤치마크 테스트 작성
- [ ] 조건부 캐시(`condition`, `unless`) 활용 사례 연구
