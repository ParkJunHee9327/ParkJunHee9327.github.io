---
layout: single
title: 트러블슈팅 - JPA/Hibernate와 jOOQ를 병행하면 뭐가 좋나요?
---

# 📌 한 줄 요약

JPA의 N+1 발생 -> ORM 기반 영속성 프레임워크 검토 이후 jOOQ 도입 <br>
* N+1 방지로 쿼리 수 1회로 축소
* 게시글의 JSONB 타입 지원 및 Converter -> 커스텀 타입 등록 및 매핑 로직 제거

<br>

👉 참고 사항
* jOOQ의 도입 전 논의 과정은 팀원과 함께 진행했으며,
* 본 글은 그 과정에서의 개인적인 고민과 구조에 대한 해석을 중심으로 정리했다.

<br>

****

# 🧐 문제: 영속성 맥락으로 인한 JPA의 N+1

## 왜 N+1 해결이 중요한가?
* 커뮤니티 웹사이트 특성 상 <u>다중 JOIN 작업 및 집계 쿼리</u>의 비중이 큼
    * ex: 사용자 + 게시글 + 댓글의 JOIN, 총 게시글 수 및 댓글 수 집계
* JPA를 사용하면 -> 부모와 연루된 자식 엔티티로 인한 N+1 발생
* 영속성 맥락으로 인한 **불필요한 데이터 조회** -> **쿼리의 비효율적 실행 위험**

``` sql
-- 사용자의 게시글에 등록된 댓글 중 isDeleted = false인 댓글 수
SELECT p.*, 
       (SELECT COUNT(*) 
        FROM comm_comment c 
        WHERE c.post_ulid = p.ulid 
          AND c.is_deleted = false) AS total_comments
FROM comm_post p
```

``` java
// JPA Repository로 치환
// N+1 발생: 게시글 수만큼 COUNT 쿼리가 추가 실행됨
public List<PostCommentCountDto> getCommentCountsPerPost() {
    List<CommPostEntity> posts = postRepository.findAll(); // 쿼리 1번

    return posts.stream()
            .map(post -> new PostCommentCountDto(
                    post.getUlid(),
                    post.getTitle(),
                    commentRepository.countByPostUlidAndIsDeletedFalse(post.getUlid()) // 쿼리 N번
            ))
            .toList();
}
```

<br>

****

# ✨ 널리 활용되는 N+1 해결책 탐구

## 영속성 프레임워크 도입 전 고려한 조건 + 애플리케이션의 필요
* **JPA N+1 방지 가능성:** 비효율적 쿼리 호출 가능성 차단
* **불필요한 데이터 로딩 방지:** 조회 및 집계 작업 시 필요한 데이터만 쿼리하기를 희망함
* **팀원들의 숙련도:** 네이티브 SQL 친숙도가 높은 상태임
* **애플리케이션의 필요:** JSONB 지원 여부(게시글 데이터 처리), 페이지네이션 적합성, 동적 쿼리(게시글 검색 등) 지원

<br>

국내에서 <u>N+1 문제 해결 방식으로 가장 널리 사용되는</u> fetch join 및 QueryDSL을 고려함

<br>

## 🔍 JPA의 fetch join
### 부합하는 점
* **N+1 해결:** 프록시 초기화 X -> 한 번의 쿼리 실행으로 부모 + 자식 데이터를 한 번에 불러옴
* **성능 개선:** 애플리케이션 계층에서 여러 번 loop하며 쿼리 실행할 필요 X -> DB 조회 비용 절감 예상
### 부합하지 않는 점
* **불필요한 데이터 로딩 가능성:** ORM 기반 작동 매커니즘 -> 엔티티 전체 조회
* **JSONB 처리 어려움:** JSONB를 Java의 POJO, Map 등과 매핑 가능 -> But 쿼리 작성 난이도 상승(네이티브 쿼리 필요)
* **페이징 처리의 위험성(1:N 관계 시):** 부모 + 자식의 모든 데이터를 메모리로 읽은 후 메모리에서 페이징 처리 -> OutOfMemoryError 유발 가능
    * OneToMany를 별도로 조회하는 등 기술적으로는 해결 가능 -> But 쿼리의 복잡도 증가

<br>

## 🔍 QueryDSL
### 부합하는 점
* **N+1 해결:** fetchJoin() 사용 -> 한 번에 데이터 조회 가능
* **동적 쿼리 용이성:** BooleanBuilder 등 런타임 맥락에 따른 쿼리 작성 용이
* **쿼리 가독성:** 메서드 체이닝 방식으로 SQL과 유사한 구조
### 부합하지 않는 점
* **팀원들의 학습 곡선:** 팀원들에게 친숙하지 않은 JPQL 기반
* **페이징 처리의 위험성:** ORM 기반 구조로 인해 fetch join과 공유하는 문제
* **부자연스러운 선택적 데이터 조회:** DTO 프로젝션 사용 시 선택적 데이터 조회 가능 -> But 엔티티 사용이 어려워 dirty checking 등 ORM의 장점 활용 X
* **JSONB 처리 우회:** 표준 SQL 위주로 지원하는 JPQL의 특성 -> 문자열 기반 템플릿 사용

<br>

## 판단 결과
* fetch join, QueryDSL의 기반인 ORM은 RDBMS와 Java 객체 매핑 + 완전한 엔티티 기반 영속성 맥락을 관리하는 설계 의도가 있음
* **ORM의 설계 의도**가 선택적 데이터 조회, 다중 JOIN 및 집계 쿼리 수행 등의 **요구사항과 다소 맞지 않음**

<br>

****

# ✨ 해결 및 성과: 왜 jOOQ인가

## 🔍 jOOQ
* **N+1 해결:** 네이티브 SQL과 유사한 구조로 쿼리 1번에 해결
* **정확한 데이터 조회:** ORM 특유의 영속성 맥락 X -> SQL에 의도된 데이터만 조회 가능
* **완만한 학습 곡선:** 네이티브 SQL과 유사한 구조 -> SQL 역량 있는 팀원들이 친숙함
* **JSONB 직접 지원:** PostgreDSL 전용 DSL 존재
* **용이한 페이지네이션:** 표준 SQL 사용 -> LIMIT, OFFSET 등 지장 X
* **간편한 동적 쿼리:** Condition 객체를 별도로 분리하여 구성 + 쿼리가 복잡해져도 jOOQ 생성 클래스로 컴파일 타입 체크

<br>

## 최종 결론: JPA/Hibernate와 jOOQ의 병행
### JPA/Hibernate
* ORM 기반 영속성 관리 및 dirty checking 이점 -> 데이터 정합성이 필수인 CUD 작업에 활용
* 완전한 엔티티를 구성해도 부담이 적은 일부 단순한 Read 작업에 활용
### jOOQ
* 영속성 맥락이 없어 필요한 데이터만 조회 가능 이점 -> 방대한 Read 작업 및 집계 쿼리에 활용
* 조건부 조회 작업에 적합한 동적 쿼리 기능 -> 다양한 조건 기반 게시글 검색에 활용

<br>

****
