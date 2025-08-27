---
layout: page
title: "AI 이벤트 조회 성능 최적화"
date: 2025-08-26
categories: [projects]
tags: [QueryDSL, 성능최적화, Spring, JPA, AI이벤트]
---

# AI 이벤트 조회 성능 최적화

## 📝 프로젝트 개요

- **기간**: 2025-08-26
- **기술스택**: Spring Boot, QueryDSL, JPA, MariaDB
- **역할**: Backend 성능 최적화
- **목적**: AI 이벤트 조회 API의 성능을 개선하여 사용자 경험 향상

## 🛠 주요 구현 사항

### 1. 기존 JPQL 방식의 문제점

```java
// 기존 코드 - 분리된 쿼리와 N+1 문제
if (userId != null) {
    List<UserSiteResponse> userSiteResponses = userSiteMapService.getSiteNameByUserId(userId);
    List<Long> siteIds = userSiteResponses.stream().map(UserSiteResponse::getSiteId).toList();
    Page<AiEvent> aiEvents = eventRepository.findEventBySiteIds(siteIds, startAt, endAt, pageable);
    return aiEvents.map(EventResponse::new);
}
if (siteId != null) {
    Page<AiEvent> aiEvents = eventRepository.findEventBySiteId(siteId, startAt, endAt, pageable);
    return aiEvents.map(this::mapToEventResponseWithObject);
}
```

**문제점:**

- 분리된 쿼리로 인한 네트워크 왕복 증가
- N+1 문제 발생 가능성
- 중복 코드와 복잡한 분기 로직

### 2. QueryDSL을 활용한 통합 최적화

```java
// 개선된 코드 - 통합 QueryDSL 메서드
@Override
public Page<EventResponse> findEventBySiteIdAndUserId(Long userId, Long siteId,
        LocalDateTime startAt, LocalDateTime endAt, Pageable pageable) {
    BooleanBuilder builder = new BooleanBuilder();

    if (startAt != null) {
        builder.and(aiEvent.eventDateTime.gt(startAt));
    }
    if (endAt != null) {
        builder.and(aiEvent.eventDateTime.lt(endAt));
    }

    if (siteId != null) {
        builder.and(site.siteId.eq(siteId));
    }

    if (userId != null) {
        builder.and(userSiteMap.user.userId.eq(userId));
    }

    // 전체 카운트 조회
    long total = queryFactory
        .selectFrom(aiEvent)
        .leftJoin(aiEvent.camera, camera)
        .leftJoin(camera.camGroup, camGroup)
        .leftJoin(camGroup.site, site)
        .leftJoin(userSiteMap).on(userSiteMap.site.eq(site))
        .where(builder)
        .fetchCount();

    // 페이징된 이벤트 조회
    List<EventResponse> eventResponses = queryFactory
        .select(new QEventResponse(aiEvent))
        .from(aiEvent)
        .leftJoin(aiEvent.camera, camera).fetchJoin()
        .leftJoin(camera.camGroup, camGroup).fetchJoin()
        .leftJoin(camGroup.site, site).fetchJoin()
        .leftJoin(site.bucket, bucket).fetchJoin()
        .leftJoin(userSiteMap).on(userSiteMap.site.eq(site))
        .where(builder)
        .orderBy(aiEvent.eventDateTime.desc())
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    // 객체 정보 일괄 조회
    List<Long> eventIds = eventResponses.stream()
        .map(EventResponse::getEventId)
        .toList();
    Map<Long, List<EventObjectResponse>> objectsMap = queryFactory
        .selectFrom(aiEventObject)
        .join(aiEventObject.aiEvent, aiEvent).fetchJoin()
        .where(aiEventObject.aiEvent.eventId.in(eventIds))
        .fetch()
        .stream()
        .collect(Collectors.groupingBy(
            obj -> obj.getAiEvent().getEventId(),
            Collectors.mapping(
                EventObjectResponse::new,
                Collectors.toList()
            )
        ));

    // EventResponse에 객체 리스트 설정
    eventResponses.forEach(eventResponse -> {
        List<EventObjectResponse> objects = objectsMap.getOrDefault(
            eventResponse.getEventId(), new ArrayList<>());
        eventResponse.setObjects(objects);
    });

    return new PageImpl<>(eventResponses, pageable, total);
}
```

### 3. 정렬 기준 변경

```java
// Controller에서 정렬 기준을 id → eventDateTime으로 변경
@PageableDefault(size = 200, sort = "eventDateTime", direction = Sort.Direction.DESC)
```

## 🚨 트러블슈팅

### 문제 상황

- **발생 시점**: AI 이벤트 조회 API 호출 시
- **현상**:
  - 평균 응답시간 221.75ms로 느림
  - 대용량 데이터에서 더욱 성능 저하
  - 사용자 경험 악화

### 원인 분석

1. **쿼리 분리로 인한 네트워크 오버헤드**

   - userId → siteIds 조회 → 이벤트 조회 (3단계)
   - DB 왕복 3-4회 발생

2. **N+1 문제 가능성**

   - 연관 엔티티 lazy loading으로 인한 추가 쿼리

3. **중복 로직과 복잡성**
   - siteId와 userId 조건에 따른 분기 처리

### 해결 과정

1. **QueryDSL 도입 검토**

   - 동적 쿼리 생성의 장점 분석
   - 기존 코드와의 호환성 검토

2. **통합 쿼리 설계**

   - LEFT JOIN을 활용한 단일 쿼리로 통합
   - fetchJoin으로 N+1 문제 해결

3. **성능 테스트 환경 구축**
   - 성능 측정 로깅 추가
   - 부하 테스트 API 개발

### 최종 해결책

**QueryDSL 기반 통합 조회 메서드**로 리팩터링

```java
// AiService 단순화
@Transactional(readOnly = true)
public Page<EventResponse> getEvents(Long siteId, Long userId,
    LocalDateTime startAt, LocalDateTime endAt, Pageable pageable) {
    long startTime = System.currentTimeMillis();

    // 날짜 기본값 설정
    if (endAt == null) endAt = LocalDateTime.now();
    if (startAt == null) startAt = endAt.minusYears(1);

    // 입력 검증
    if ((userId == null && siteId == null) || (userId != null && siteId != null)) {
        throw new BusinessException(ErrorCode.INVALID_INPUT_VALUE);
    }

    // 통합 쿼리 호출
    Page<EventResponse> result = eventRepository.findEventBySiteIdAndUserId(
        userId, siteId, startAt, endAt, pageable);

    // 성능 측정 로깅
    long endTime = System.currentTimeMillis();
    log.info("getEvents 성능 측정 - userId: {}, siteId: {}, 실행시간: {}ms, 결과수: {}",
        userId, siteId, (endTime - startTime), result.getContent().size());

    return result;
}
```

### 배운 점

1. **QueryDSL의 강력함**

   - 동적 쿼리 생성의 편리함
   - 타입 안전성과 가독성 확보

2. **성능 최적화의 중요성**

   - 단일 지표로 71.6% 성능 향상 달성
   - 사용자 경험에 직접적 영향

3. **체계적인 성능 테스트**
   - 단계별 부하 증가로 최적점 발견
   - 시스템 한계점 파악의 중요성

## 📊 성능 개선 결과

### 단일 요청 성능 비교

| 지표              | 기존 (JPQL) | 개선 (QueryDSL) | 개선도         |
| ----------------- | ----------- | --------------- | -------------- |
| **평균 응답시간** | 221.75ms    | 62.95ms         | **71.6% 감소** |
| **최소 응답시간** | 134ms       | 44ms            | **67.2% 감소** |
| **최대 응답시간** | 503ms       | 182ms           | **63.8% 감소** |

### 부하 테스트 결과

| 단계      | 요청수  | 스레드 | 실행시간  | 평균응답시간 | TPS        |
| --------- | ------- | ------ | --------- | ------------ | ---------- |
| 1단계     | 50      | 5      | 908ms     | 90.42ms      | 55.07      |
| **2단계** | **100** | **10** | **573ms** | **53.40ms**  | **174.52** |
| 3단계     | 300     | 20     | 3,271ms   | 201.26ms     | 91.72      |

**최적 성능 구간**: **10개 스레드, 174.52 TPS**

## 🔗 참고 자료

- [QueryDSL 공식 문서](http://querydsl.com/static/querydsl/latest/reference/html/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
- [JVM 성능 튜닝 가이드](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)
