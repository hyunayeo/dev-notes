---
layout: page
title: "2025-09-04 - JPA 삭제 최적화와 ROI 서비스 구현"
date: 2025-09-04
categories: [til]
tags: [jpa, spring, optimization, bulk-delete, roi]
---

# 2025-09-04 - JPA 삭제 최적화와 ROI 서비스 구현

## 📚 새로 배운 것

### JPA 기본 삭제 동작의 문제점

JPA에서는 기본적으로 **개별 삭제**로 동작한다:

```java
// deleteAll() 내부 동작
roiRegionRepository.deleteAll(regionsToDelete);

// 실제로는 이렇게 실행됨:
// 1. SELECT * FROM roi_regions WHERE ...
// 2. DELETE FROM roi_vertices WHERE id = 1;
// 3. DELETE FROM roi_vertices WHERE id = 2;
// 4. DELETE FROM roi_vertices WHERE id = 3;
// ... (각각 개별 쿼리)
```

**이유:**

- 영속성 컨텍스트 관리
- 이벤트 리스너(@PreRemove, @PostRemove) 실행
- 연관관계 처리
- 감사(Audit) 로그

### 벌크 삭제로 최적화

```java
@Modifying
@Query("DELETE FROM RoiVertex v WHERE v.roiRegion IN :roiRegions")
void deleteByRoiRegionIn(@Param("roiRegions") List<RoiRegion> roiRegions);
```

**성능 비교:**

- 개별 삭제: N개의 DELETE 쿼리
- 벌크 삭제: 1개의 DELETE 쿼리

### N+1 문제 해결

```java
// 문제가 있는 코드
List<RoiRegion> regions = roiRegionRepository.findByAnalysis_ChannelId(channelId);
// vertices 접근 시마다 추가 쿼리 발생

// 해결책: JOIN FETCH 사용
@Query("SELECT r FROM RoiRegion r JOIN FETCH r.vertices WHERE r.analysis.channelId = :channelId")
List<RoiRegion> findByAnalysis_ChannelId(Integer channelId);
```

### 더티체크를 통한 불필요한 쿼리 방지

```java
private boolean updateRoiRegionData(RoiRegion roiRegion, RoiRegionCreateDto dto) {
    boolean hasChanges = false;

    if (!Objects.equals(roiRegion.getLineColor(), dto.getLineColor())) {
        roiRegion.setLineColor(dto.getLineColor());
        hasChanges = true;
    }

    return hasChanges; // 변경사항이 있을 때만 true 반환
}

// 변경사항이 있는 경우에만 DB 저장
if (updateRoiRegionData(existingRegion, dto)) {
    regionsToUpdate.add(existingRegion);
}
```

### 배치 처리 패턴

```java
// 효율적인 배치 처리
List<RoiRegion> regionsToSave = new ArrayList<>();
List<RoiRegion> regionsToUpdate = new ArrayList<>();
List<RoiRegion> regionsToDelete = new ArrayList<>();

// ... 로직 처리 후

// 한 번에 배치 처리
roiVertexRepository.deleteByRoiRegionIn(regionsToDelete); // 벌크 삭제
roiRegionRepository.deleteAll(regionsToDelete);
roiRegionRepository.saveAll(regionsToSave);
roiRegionRepository.saveAll(regionsToUpdate);
```

## 🤔 궁금한 점

- JPA Batch Insert/Update 설정이 성능에 어떤 영향을 줄까?
- `@Modifying(clearAutomatically = true)` 옵션을 사용해야 하는 경우는?
- 벌크 삭제 시 영속성 컨텍스트와 동기화 문제는 어떻게 해결할까?
- Soft Delete vs Hard Delete 성능 차이는?

## 📝 내일 해볼 것

- [ ] JPA Batch 설정 최적화 (hibernate.jdbc.batch_size)
- [ ] QueryDSL을 사용한 동적 벌크 연산 구현
- [ ] 실제 SQL 로그 분석해서 최적화 전후 비교
- [ ] 대용량 데이터에서의 성능 테스트
- [ ] Spring Data JPA의 `@Modifying` 어노테이션 옵션들 실험
