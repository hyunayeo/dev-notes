---
layout: page
title: "2025-09-02 - JPA Cascade 삭제 처리"
date: 2025-09-02
categories: [til]
tags: [JPA, Spring Boot, Cascade, 연관관계]
---

# 2025-09-03 - JPA Cascade 삭제 처리

## 📚 새로 배운 것

### JPA Cascade 삭제와 명시적 삭제의 차이점

- **Cascade 삭제**: 부모 엔티티가 삭제될 때만 자동으로 자식 엔티티 삭제

  ```java
  @OneToMany(mappedBy = "analysis", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<RoiRegion> roiRegions = new ArrayList<>();
  ```

- **명시적 삭제 필요한 경우**: 부모 엔티티를 삭제하지 않고 초기화(reset)할 때
  ```java
  public void reset() {
      this.camId = null;
      this.camProfileInfoId = null;
      this.aiServerAiClassType = null;
      this.aiValidScore = null;
      this.roiRegions.clear(); // 명시적으로 연관 데이터 삭제
  }
  ```

### 비즈니스 로직에서의 연관 데이터 관리

- AI 분석 타입이 변경될 때 기존 ROI 영역을 삭제해야 함
  ```java
  // AiType이 변경되면 기존 ROI 삭제
  if (!aiAnalysis.getAiServerAiClassType().equals(request.getAnalysisAiType())) {
      aiAnalysis.getRoiRegions().clear();
  }
  ```

## 🤔 궁금한 점

- orphanRemoval = true와 cascade = CascadeType.ALL의 정확한 동작 차이점
- clear() 호출 시 DB 쿼리가 어떻게 생성되는지
- 대용량 데이터에서 cascade 삭제 성능 최적화 방법

## 📝 내일 해볼 것

- JPA cascade 관련 테스트 코드 작성
- orphanRemoval 옵션 동작 확인
- 삭제 쿼리 로그 확인해보기
