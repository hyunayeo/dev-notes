---
layout: page
title: "2025-08-22 - @SpringBootTest vs @ExtendWith(MockitoExtension.class) 선택 기준"
date: 2025-08-22
categories: [til]
tags: [spring, testing, mockito, unit-test, integration-test, cache]
---

# 2025-08-22 - @SpringBootTest vs @ExtendWith(MockitoExtension.class) 선택 기준

## 📚 새로 배운 것

### 테스트 어노테이션별 특징

#### @ExtendWith(MockitoExtension.class) - 단위 테스트

```java
@ExtendWith(MockitoExtension.class)
class UserServiceUnitTest {
    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    // Spring 컨텍스트 없이 순수 로직만 테스트
}
```

#### @SpringBootTest - 통합 테스트

```java
@SpringBootTest
class UserServiceCacheTest {
    @Autowired
    private UserService userService;

    @Autowired
    private CacheManager cacheManager;

    @MockBean  // Spring 컨텍스트 내에서 Mock 사용
    private UserRepository userRepository;
}
```

### 캐시 테스트에서 SpringBootTest가 필수인 이유

#### AOP 프록시 의존성

Spring Cache는 AOP 기반으로 동작:

```java
@Cacheable(value = "userByUid", key = "#id")
public User findByUserUid(String id) {
    // 실제 메서드 호출 전/후에 캐시 로직이 AOP로 처리됨
}
```

**MockitoExtension 환경:**

- Spring 컨텍스트 없음 → AOP 프록시 없음 → 캐시 동작 안 함
- `@Cacheable` 어노테이션이 무시됨

**SpringBootTest 환경:**

- Spring 컨텍스트 있음 → AOP 프록시 생성 → 캐시 정상 동작

#### 실제 캐시 상태 검증 필요

```java
@Test
void testCacheEviction() {
    // 캐시에 데이터 저장
    userService.findByUserUid("test");

    // 실제 CacheManager를 통해 캐시 상태 확인
    assertThat(cacheManager.getCache("userByUid").get("test")).isNotNull();

    // 캐시 무효화 작업
    userService.changeUser(1L, request);

    // 캐시가 삭제되었는지 확인
    assertThat(cacheManager.getCache("userByUid").get("test")).isNull();
}
```

### 테스트 방식별 비교표

| 구분                | MockitoExtension    | SpringBootTest        |
| ------------------- | ------------------- | --------------------- |
| **Spring 컨텍스트** | ❌ 없음             | ✅ 있음               |
| **AOP/프록시**      | ❌ 없음             | ✅ 있음               |
| **캐시 동작**       | ❌ 안 됨            | ✅ 됨                 |
| **실행 속도**       | ✅ 빠름 (ms 단위)   | ❌ 느림 (초 단위)     |
| **테스트 범위**     | 단위 테스트         | 통합 테스트           |
| **의존성 주입**     | @Mock, @InjectMocks | @Autowired, @MockBean |
| **격리성**          | ✅ 높음             | ❌ 낮음               |

### 실제 테스트 동작 차이

#### MockitoExtension으로 캐시 테스트 시도 (실패 케이스)

```java
@ExtendWith(MockitoExtension.class)
class FailedCacheTest {
    @Test
    void testCacheHit() {
        userService.findByUserUid("test");  // 첫 호출
        userService.findByUserUid("test");  // 두 번째 호출

        // 캐시가 동작하지 않으므로 2번 모두 DB 호출됨
        verify(userRepository, times(2)).findByUserUid("test"); // 예상과 다름!
    }
}
```

#### SpringBootTest로 캐시 테스트 (성공 케이스)

```java
@SpringBootTest
class SuccessfulCacheTest {
    @Test
    void testCacheHit() {
        userService.findByUserUid("test");  // DB 호출 + 캐시 저장
        userService.findByUserUid("test");  // 캐시에서 반환 (DB 호출 안 함)

        // 캐시가 정상 동작하므로 DB는 1번만 호출
        verify(userRepository, times(1)).findByUserUid("test"); // 예상대로!
    }
}
```

### 테스트 전략 가이드

#### 단위 테스트용 - MockitoExtension

```java
@ExtendWith(MockitoExtension.class)
class UserServiceBusinessLogicTest {
    // 순수 비즈니스 로직 테스트
    // 예: 유효성 검사, 데이터 변환, 예외 처리 등
}
```

#### 통합 테스트용 - SpringBootTest

```java
@SpringBootTest
class UserServiceIntegrationTest {
    // Spring 기능이 포함된 테스트
    // 예: 캐시, 트랜잭션, AOP, 의존성 주입 등
}
```

### 성능 최적화 팁

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("test")
class OptimizedIntegrationTest {
    // webEnvironment.NONE: 웹 서버 시작 안 함 → 속도 향상
    // ActiveProfiles("test"): 테스트용 설정 사용
}
```

## 🤔 궁금한 점

- @SpringBootTest에서 특정 설정만 로드하여 속도를 개선하는 방법은?
- @TestConfiguration을 활용한 테스트용 캐시 설정 최적화 방법은?
- Slice Test(@WebMvcTest, @DataJpaTest)와 캐시 테스트 조합은 가능할까?
- TestContainers를 사용한 실제 Redis 캐시 테스트 전략은?

## 📝 내일 해볼 것

- [ ] @TestConfiguration으로 테스트용 캐시 설정 분리
- [ ] Slice Test에서 캐시 기능 테스트 방법 연구
- [ ] 캐시 성능 테스트를 위한 벤치마크 코드 작성
- [ ] Redis 캐시와 TestContainers 조합 테스트 구현
- [ ] 테스트 실행 시간 최적화 방법 실험
