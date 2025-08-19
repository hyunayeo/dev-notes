---
layout: page
title: "[API] POST 요청 중복 등록 방지"
date: 2025-08-19
categories: [troubleshooting, api]
tags: [bug, api, duplicate, idempotency, database]
---

# [API] POST 요청 중복 등록

**발생일:** 2025-08-19  
**프로젝트:** WebVMS  
**카테고리:** API

## 🚨 문제 상황

사용자 등록, 지도 설정 등록 등 POST API에서 동일한 요청이 여러 번 전송될 때 중복 데이터가 생성되는 문제가 발생함

**증상:**

- 사용자가 등록 버튼을 연속으로 클릭 시 동일한 데이터가 여러 개 생성됨
- 네트워크 지연으로 인한 재요청 시 중복 등록
- 모바일 환경에서 더블탭으로 인한 중복 호출
- 브라우저 새로고침이나 뒤로가기 후 재전송으로 인한 중복

**현재 MapController 예시:**

```java
@PostMapping
public DefaultResponse<String> createMapConfig(@RequestBody @Valid MapCreateRequest request) {
    mapService.create(request);
    return DefaultResponse.ofSuccess();
}
```

## 🔍 원인 분석

1. **네트워크 레벨**

   - 클라이언트의 재시도 로직
   - 타임아웃으로 인한 중복 요청

2. **사용자 행동**

   - 응답 지연 시 버튼 연속 클릭
   - 브라우저 뒤로가기 후 폼 재전송
   - 모바일 더블탭 제스처

3. **애플리케이션 레벨**
   - 중복 방지 로직 부재
   - 트랜잭션 처리 미흡
   - 유니크 제약 조건 부재

## 🛠 해결 과정

### 1. 데이터베이스 유니크 제약 조건 (기본적 방법)

**장점:** 간단하고 확실한 중복 방지  
**단점:** 중복 시 예외 발생으로 사용자 경험 저하

```sql
-- 사용자 테이블에 이메일 유니크 제약 추가
ALTER TABLE users ADD CONSTRAINT uk_user_email UNIQUE (email);

-- 복합 유니크 제약 (사이트별 지도 설정)
ALTER TABLE maps ADD CONSTRAINT uk_site_map UNIQUE (site_id, map_name);
```

```java
@Entity
@Table(name = "users")
public class User {
    @Column(unique = true, nullable = false)
    private String email;

    // 복합 유니크 제약
    @Table(uniqueConstraints = {
        @UniqueConstraint(columnNames = {"site_id", "map_name"})
    })
}
```

### 2. 서비스 레벨 중복 체크 (개선된 방법)

**장점:** 사용자 친화적 에러 처리  
**단점:** 동시성 문제 가능성

```java
@Service
@Transactional
public class MapService {

    public void create(MapCreateRequest request) {
        // 중복 체크
        if (mapRepository.existsBySiteIdAndMapName(request.getSiteId(), request.getMapName())) {
            throw new DuplicateResourceException("이미 존재하는 지도 설정입니다.");
        }

        // 저장
        Map map = Map.builder()
            .siteId(request.getSiteId())
            .mapName(request.getMapName())
            .config(request.getConfig())
            .build();

        mapRepository.save(map);
    }
}

// Repository 메서드
public interface MapRepository extends JpaRepository<Map, Long> {
    boolean existsBySiteIdAndMapName(Long siteId, String mapName);
}
```

### 3. Idempotency Key 패턴 (권장 방법)

**장점:** 완벽한 중복 방지 및 재시도 안전성  
**단점:** 구현 복잡도 증가

#### 3-1. Idempotency 테이블 생성

```sql
CREATE TABLE idempotency_keys (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    idempotency_key VARCHAR(255) NOT NULL UNIQUE,
    request_hash VARCHAR(255) NOT NULL,
    response_data JSON,
    status VARCHAR(20) NOT NULL DEFAULT 'PROCESSING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    INDEX idx_key_expires (idempotency_key, expires_at)
);
```

#### 3-2. Entity 및 Repository

```java
@Entity
@Table(name = "idempotency_keys")
public class IdempotencyKey {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String idempotencyKey;

    @Column(nullable = false)
    private String requestHash;

    @Column(columnDefinition = "JSON")
    private String responseData;

    @Enumerated(EnumType.STRING)
    private IdempotencyStatus status;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime expiresAt;
}

public enum IdempotencyStatus {
    PROCESSING, COMPLETED, FAILED
}

@Repository
public interface IdempotencyKeyRepository extends JpaRepository<IdempotencyKey, Long> {
    Optional<IdempotencyKey> findByIdempotencyKeyAndExpiresAtAfter(
        String idempotencyKey, LocalDateTime now
    );
}
```

#### 3-3. Idempotency 서비스

```java
@Service
@Transactional
public class IdempotencyService {
    private final IdempotencyKeyRepository repository;
    private final ObjectMapper objectMapper;

    public <T> T executeIdempotent(String idempotencyKey, String requestHash,
                                   Supplier<T> operation, Class<T> responseType) {

        // 기존 키 확인
        Optional<IdempotencyKey> existing = repository
            .findByIdempotencyKeyAndExpiresAtAfter(idempotencyKey, LocalDateTime.now());

        if (existing.isPresent()) {
            IdempotencyKey key = existing.get();

            // 요청 내용이 다른 경우
            if (!key.getRequestHash().equals(requestHash)) {
                throw new IdempotencyConflictException(
                    "동일한 Idempotency Key로 다른 요청이 전송되었습니다."
                );
            }

            // 처리 중인 경우
            if (key.getStatus() == IdempotencyStatus.PROCESSING) {
                throw new IdempotencyProcessingException(
                    "요청이 처리 중입니다. 잠시 후 다시 시도해주세요."
                );
            }

            // 완료된 경우 기존 응답 반환
            if (key.getStatus() == IdempotencyStatus.COMPLETED) {
                return deserializeResponse(key.getResponseData(), responseType);
            }
        }

        // 새 키 생성
        IdempotencyKey newKey = IdempotencyKey.builder()
            .idempotencyKey(idempotencyKey)
            .requestHash(requestHash)
            .status(IdempotencyStatus.PROCESSING)
            .expiresAt(LocalDateTime.now().plusHours(24))
            .build();

        repository.save(newKey);

        try {
            // 실제 작업 수행
            T result = operation.get();

            // 성공 시 결과 저장
            newKey.setResponseData(serializeResponse(result));
            newKey.setStatus(IdempotencyStatus.COMPLETED);
            repository.save(newKey);

            return result;

        } catch (Exception e) {
            // 실패 시 상태 업데이트
            newKey.setStatus(IdempotencyStatus.FAILED);
            repository.save(newKey);
            throw e;
        }
    }

    private String calculateHash(Object request) {
        try {
            String json = objectMapper.writeValueAsString(request);
            return DigestUtils.sha256Hex(json);
        } catch (Exception e) {
            throw new RuntimeException("요청 해시 계산 실패", e);
        }
    }
}
```

#### 3-4. Controller 적용

```java
@RestController
@RequestMapping({"/api/v1/maps", "/api/v1/admin/maps"})
public class MapController {
    private final MapService mapService;
    private final IdempotencyService idempotencyService;

    @PostMapping
    public DefaultResponse<String> createMapConfig(
            @RequestBody @Valid MapCreateRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {

        // Idempotency Key 검증
        if (StringUtils.isBlank(idempotencyKey)) {
            throw new BadRequestException("Idempotency-Key 헤더가 필요합니다.");
        }

        // 중복 방지 실행
        String result = idempotencyService.executeIdempotent(
            idempotencyKey,
            calculateRequestHash(request),
            () -> {
                mapService.create(request);
                return "Map created successfully";
            },
            String.class
        );

        return DefaultResponse.ofSuccess(result);
    }

    private String calculateRequestHash(Object request) {
        // ObjectMapper를 사용한 해시 계산
        try {
            String json = objectMapper.writeValueAsString(request);
            return DigestUtils.sha256Hex(json);
        } catch (Exception e) {
            throw new RuntimeException("요청 해시 계산 실패", e);
        }
    }
}
```

#### 3-5. 클리이언트 구현 예시

```javascript
// 프론트엔드에서 Idempotency Key 생성
function generateIdempotencyKey() {
  return "key_" + Date.now() + "_" + Math.random().toString(36).substr(2, 9);
}

// API 호출
async function createMap(mapData) {
  const idempotencyKey = generateIdempotencyKey();

  try {
    const response = await fetch("/api/v1/maps", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(mapData),
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return await response.json();
  } catch (error) {
    console.error("지도 생성 실패:", error);
    throw error;
  }
}
```

#### 3-6. 예외 처리

```java
// 커스텀 예외 클래스들
@ResponseStatus(HttpStatus.CONFLICT)
public class IdempotencyConflictException extends RuntimeException {
    public IdempotencyConflictException(String message) {
        super(message);
    }
}

@ResponseStatus(HttpStatus.TOO_MANY_REQUESTS)
public class IdempotencyProcessingException extends RuntimeException {
    public IdempotencyProcessingException(String message) {
        super(message);
    }
}

// 글로벌 예외 핸들러
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IdempotencyConflictException.class)
    public ResponseEntity<DefaultResponse<String>> handleIdempotencyConflict(
            IdempotencyConflictException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(DefaultResponse.ofFailure(ex.getMessage()));
    }

    @ExceptionHandler(IdempotencyProcessingException.class)
    public ResponseEntity<DefaultResponse<String>> handleIdempotencyProcessing(
            IdempotencyProcessingException ex) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .body(DefaultResponse.ofFailure(ex.getMessage()));
    }
}
```

### 4. 정리 및 최종 해결책

**권장 구현 순서:**

1. **1단계:** DB 유니크 제약 조건 추가 (필수)
2. **2단계:** 서비스 레벨 중복 체크 추가 (기본)
3. **3단계:** Idempotency Key 패턴 도입 (고도화)

**적용된 MapController 최종 버전:**

```java
@PostMapping
public DefaultResponse<String> createMapConfig(
        @RequestBody @Valid MapCreateRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {

    if (StringUtils.isNotBlank(idempotencyKey)) {
        // Idempotency 패턴 사용
        String result = idempotencyService.executeIdempotent(
            idempotencyKey,
            calculateRequestHash(request),
            () -> {
                mapService.create(request);
                return "Map configuration created successfully";
            },
            String.class
        );
        return DefaultResponse.ofSuccess(result);
    } else {
        // 기본 중복 체크
        mapService.create(request);
        return DefaultResponse.ofSuccess("Map configuration created successfully");
    }
}
```

## 💡 배운 점

1. **다층 방어의 중요성**

   - DB 제약 조건, 서비스 로직, API 패턴을 조합하여 완벽한 중복 방지
   - 각 레벨마다 다른 장단점이 있어 상황에 맞는 선택 필요

2. **Idempotency의 핵심 개념**

   - 동일한 요청을 여러 번 수행해도 결과가 같아야 함
   - 클라이언트가 안전하게 재시도할 수 있는 환경 제공
   - Stripe, PayPal 등 결제 API에서 필수적으로 사용되는 패턴

3. **사용자 경험 고려사항**

   - 단순 에러 응답보다는 명확한 메시지와 처리 방법 제시
   - 프론트엔드에서의 버튼 비활성화와 서버 측 검증의 조합
   - 네트워크 지연 상황에 대한 적절한 사용자 피드백

4. **성능 최적화**

   - Idempotency Key 테이블의 적절한 TTL 설정
   - 인덱스 최적화로 조회 성능 향상
   - 만료된 키들에 대한 정리 작업 스케줄링

5. **보안 고려사항**
   - Idempotency Key의 추측 불가능성
   - 요청 해시를 통한 내용 변조 방지
   - 적절한 만료 시간 설정으로 저장 공간 최적화

## 🔗 참고 자료

- [Stripe API - Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [HTTP Idempotency Key Header Field (Draft RFC)](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header)
- [Spring Boot - Data Validation](https://spring.io/guides/gs/validating-form-input/)
- [Spring Data JPA - Unique Constraints](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.entity-persistence.saving-entities)
- [PayPal Developer - Idempotency](https://developer.paypal.com/api/rest/idempotency/)
- [AWS API Gateway - Handling Duplicate API Requests](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-idempotency.html)
- [REST API Best Practices - Idempotency](https://restfulapi.net/idempotent-rest-apis/)
