---
layout: page
title: "2025-09-12 - JPA Cascade & 코드 리팩터링"
date: 2025-09-12
categories: [til]
tags: [JPA, Spring Boot, 리팩터링, Cascade]
---

# 2025-09-12 - JPA Cascade & 코드 리팩터링

## 📚 새로 배운 것

### JPA Cascade Operations와 orphanRemoval

- `@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)` 설정을 활용하면 복잡한 배치 처리 로직을 대폭 간소화할 수 있다
- `orphanRemoval = true`: 부모 엔티티에서 제거된 자식 엔티티가 DB에서 자동 삭제
- `CascadeType.ALL`: 부모 저장 시 자식 엔티티들도 자동으로 저장/업데이트

**Before (복잡한 배치 처리)**:
```java
// 기존 데이터 조회 및 분석
List<RoiRegion> existingRegions = roiRegionRepository.findByAnalysis_ChannelId(channelId);
// 삭제/수정/생성 분류 로직...
roiRegionRepository.deleteAll(regionsToDelete);
roiRegionRepository.saveAll(allToSave);
```

**After (JPA Cascade 활용)**:
```java
aiAnalysis.getRoiRegions().clear();
aiAnalysis.getRoiRegions().addAll(newRegions);
aiAnalysisRepository.save(aiAnalysis);  // 한 번의 save로 모든 CRUD 처리
```

### 메서드 모듈화와 코드 중복 제거

- 공통 로직을 별도 메서드로 추출하여 코드 중복을 약 30줄 제거
- `validateAndGetAIAnalysis()`: 유효성 검증 로직 분리
- `processAnalysis(boolean isUpdate)`: 플래그를 활용한 create/update 차이점 처리

```java
private void processAnalysis(AIAnalysis aiAnalysis, AnalysisUpdateRequest request, boolean isUpdate) {
    if (isUpdate && !aiAnalysis.getAiServerAiClassType().equals(request.getAnalysisAiType())) {
        aiAnalysis.getRoiRegions().clear();  // AI 타입 변경 시에만 ROI 삭제
    }
    // 공통 처리 로직...
}
```

## 🤔 궁금한 점

- JPA Cascade 작업과 수동 배치 처리 중 어느 것이 성능상 더 유리한가?
- `CascadeType.ALL`과 개별 cascade 타입 지정의 성능 차이는?
- `orphanRemoval = true`가 대량의 자식 엔티티에서도 효율적으로 작동하는가?

## 📝 내일 해볼 것

- JPA Cascade 성능 테스트 (대량 데이터 환경에서)
- `@EntityGraph`를 활용한 N+1 문제 해결 방법 탐구
- 코드리뷰에서 발견된 deleteAnalysis 메서드의 문제점 수정
- JPA Batch Insert/Update 최적화 방법 연구