---
layout: single
title: 트러블슈팅 - 데이터 무결성과 복잡한 쿼리의 효율적 실행을 위한 JPA와 jOOQ
---

# 📌 요약본

* JPA의 N+1 문제와 JSONB 처리 한계를 해결하기 위해 기술 선택을 검토함
* QueryDSL 역시 ORM 기반으로 동일한 구조적 제약(페이징, JSONB, fetch join 한계)을 공유한다고 판단함
* 결론적으로 CUD는 JPA/Hibernate, 복잡한 조회·집계 쿼리는 jOOQ로 분리하는 구조 설계
* N+1 쿼리를 단일 쿼리로 개선하고 JSONB를 직접 제어 가능한 구조로 구축하는 성과 이룩

<br>

👉 참고 사항
* jOOQ의 도입 전 논의 과정은 팀원과 함께 진행했으며,
* 본 글은 그 과정에서의 개인적인 고민과 구조에 대한 해석을 중심으로 정리했다.

<br>

****

# 🧐 문제: JPA의 N+1 문제

## 왜 N+1 해결이 중요한가?
* 커뮤니티 웹사이트 특성 상 <u>다중 읽기 작업 및 집계 쿼리</u>의 비중이 큼
    * ex: 사용자 + 게시글 + 댓글의 JOIN에서 읽기, 총 게시글 수 및 댓글 수 집계
* JPA의 기본적인 지연 로딩 전략과 연관관계 탐색 방식은 데이터 무결성에 중요
    * But 이런 상태에서 대량 조회 또는 집계 쿼리를 실행할 경우에는 **불필요한 데이터 조회** -> **쿼리의 비효율적 실행 위험**
    * *JPA의 문제가 아님.* 다만, ORM 기반 객체 탐색 방식이 조회/집계 쿼리와 맞지 않는 구조적인 현상

``` java
// N+1 예시
List<Order> orders = orderRepository.findByStatusAndCreatedAtAfter(
    OrderStatus.PAID, 
    LocalDateTime.now().minusDays(7)
);

// N+1 두 번 발생
for (Order order : orders) {
    // (쿼리 N) SELECT * FROM order_items WHERE order_id = ?
    int totalQty = order.getOrderItems().stream()
        .mapToInt(OrderItem::getQuantity)
        .sum();

    // (쿼리 N) SELECT * FROM coupons WHERE id = ?
    String couponName = order.getCoupon() != null 
        ? order.getCoupon().getName() 
        : "없음";

    log.info("주문 {} / 총 수량: {} / 쿠폰: {}", order.getId(), totalQty, couponName);
}
```

<br>

****

# ✨ 의사결정 과정: ORM 영속성 프레임워크 기반 N+1 해결책 탐구

## 영속성 프레임워크 도입 전 고려한 조건 + 애플리케이션의 필요

### 성능 및 쿼리 요구사항

* **JPA N+1 방지 가능성:** 비효율적 쿼리 호출 가능성 차단
* **불필요한 데이터 로딩 최소화:** 조회 및 집계 작업 시 필요한 데이터만 쿼리하기를 희망함

### 데이터베이스 활용 요구

* **JSONB 직접 지원 여부:** 게시글 데이터 처리
* **동적 쿼리:** 게시글 검색 등
* **페이지네이션 안정성:** 게시글 및 댓글 조회 시 필요

### 팀 상황

* **팀원들의 숙련도:** 네이티브 SQL 친숙도가 높은 상태임

<br>

국내에서 <u>N+1 문제 해결 방식으로 가장 널리 사용되는</u> fetch join 및 QueryDSL을 고려함

<br>

## 🔍 JPA의 fetch join

### 부합하는 점

* **N+1 완화:** 연관된 엔티티를 즉시 로딩 -> 한 번의 쿼리 실행으로 부모 + 자식 데이터를 한 번에 불러옴
* **성능 개선:** 애플리케이션 계층에서 여러 번 loop하며 쿼리 실행할 필요 X -> DB 조회 비용 절감 예상

### 부합하지 않는 점

* **불필요한 데이터 로딩 가능성:** ORM 기반 작동 매커니즘 -> 컬럼 단위의 제어 어려움
    * DTO Projection으로 필요한 데이터 조회 가능 -> 엔티티 기반 dirty checking 등 이점과의 trade off
* **간접적인 JSONB 처리:** JSONB를 Java의 POJO, Map 등과 매핑 가능 -> But 쿼리 작성 난이도 상승(네이티브 쿼리 필요)
* **페이징 처리의 위험성(1:N 관계 시):** JOIN으로 인해 부모 row가 중복 생성 -> JPA가 쿼리 결과를 후처리함 -> 정확한 페이징 어려움 + 성능 저하
    * OneToMany를 별도로 조회하는 등 기술적으로는 해결 가능 -> But 쿼리의 복잡도 증가

<br>

## 🔍 QueryDSL

### 부합하는 점

* **N+1 완화:** fetchJoin() 사용 -> 한 번에 데이터 조회 가능
* **동적 쿼리 용이성:** BooleanBuilder 등 런타임 맥락에 따른 쿼리 작성 용이
* **쿼리 가독성:** 메서드 체이닝 방식으로 쿼리 작성의 깔끔함

### 부합하지 않는 점

* **페이징 처리의 위험성:** ORM 기반 구조로 인해 fetch join과 공유하는 문제
* **부자연스러운 선택적 데이터 조회:** DTO 프로젝션 사용 시 선택적 데이터 조회 가능 -> But 엔티티 사용이 어려워 dirty checking 등 ORM의 장점 활용 X
* **JSONB 처리 우회:** 표준 SQL 위주로 지원하는 JPQL의 특성 -> stringTemplate 등 DB 특화 기능을 우회하여 사용
* **팀원들의 학습 곡선:** 팀원들이 네이티브 SQL에 친숙함 + 개발 일정 등 현실적 시간 제약 -> JPQL 학습에 부담

<br>

## 잠정적 가설

* ORM 기반 영속성 프레임워크 사용시 일반적인 CRUD 기능의 경우 JPA만으로 충분한 성능 확보 가능
    * But 현재 프로젝트의 맥락에서는 다중 JOIN 및 집계 쿼리 비중이 높음 -> 쿼리에 대한 제어권이 더욱 필요
* ORM은 객체 중심의 데이터 접근 및 영속성 기반 데이터 무결성 유지의 이점이 있음
    * But 다중 JOIN, 집계 중심의 쿼리에서는 SQL을 직접 제어하는 방식이 더 적합할 수 있다는 가설을 세움
* **결론:** ORM 기반 방식은 충분히 강점이 있는 방식임
    * But 다중 JOIN + 집계 중심 환경, DB 전용 기능의 활용도(게시글의 JSONB)에서는 <u>SQL 직접 제어가 더 적합할 수 있음</u>

<br>

****

# ✨ 결론 및 성과: JPA/Hibernate + jOOQ 병행

## 🔍 jOOQ

* **N+1 방지:** ORM에서 자유로움 + SQL 제어권이 높아 N+1이 발생하지 않도록 통제 가능
* **정확한 데이터 조회:** 영속성 컨텍스트에 의존하지 않고 필요한 컬럼만 선택적으로 조회
* **완만한 학습 곡선:** 팀원들이 SQL에 익숙함 -> 기존 쿼리 작성 방식과 유사한 흐름으로 적응 가능
* **JSONB 직접 지원:** PostgreDSL 전용 DSL 존재
* **용이한 페이지네이션:** 표준 SQL 사용 -> LIMIT, OFFSET 등 지장 X
* **간편한 동적 쿼리:** Condition 객체를 별도로 분리하여 구성 + 쿼리가 복잡해져도 jOOQ 생성 클래스로 컴파일 타입 체크

<br>

## 가설의 검증 과정

* 실제 서비스 코드 수준의 성능 비교 실험은 진행하지 않았음. 공식 문서 및 여러 자료를 기반으로 쿼리 표현력, JSONB 지원 방식, 개발 생산성 관점에서 비교 검토를 진행함
* 개발 일정이 있는 현실적 제약 속에서 문헌/구조 기반으로 판단하였으므로 검증의 배경을 인지할 필요가 있음

> 결론: jOOQ는 JSONB를 DB 타입 그대로 다룰 수 있는 반면, JPA/QueryDSL은 문자열 기반으로 우회 처리해야 했다.

<br>

## JSONB 처리 (post의 content가 JSONB 타입)

* 아래 코드는 공식 문서 및 예제를 기반으로 재구성한 예시임
* 실제 프로젝트에서는 jOOQ 기반 커스텀 Converter를 별도로 구현하여 사용했음

**jOOQ**

``` java
public class JsonNodeConverter implements Converter<JSONB, JsonNode> {
    @Override
    public JsonNode from(JSONB json) { // JSONB <-> JsonNode 직접 변환
        return new ObjectMapper().readTree(json.data());
    }

    @Override
    public JSONB to(JsonNode node) { // JsonNode <-> JSONB 직접 변환
        return JSONB.valueOf(node.toString());
    }

    @Override public Class<JSONB>     fromType() { return JSONB.class; }
    @Override public Class<JsonNode>  toType()   { return JsonNode.class; }
}
```

``` sql
// JSONB가 JsonNode로 자동 변환됨
List<Post> posts = dsl
    .selectFrom(POSTS)
    .where(POSTS.CONTENT.cast(String.class).contains("author"))
    .fetchInto(Post.class);

// JSONB 연산자도 직접 사용 가능
dsl.select(POSTS.CONTENT.field("title", String.class))
   .from(POSTS)
   .fetch();
```

**QueryDSL**

``` java
@Entity
@Table(name = "posts")
public class Post {

    @Id Long id;

    @Convert(converter = JsonNodeConverter.class)
    @Column(name = "content", columnDefinition = "jsonb")
    private JsonNode content;  // JPA가 String <-> JsonNode 변환
}
```

``` java
// AttributeConverter로 구현
@Converter(autoApply = true)
public class JsonNodeConverter 

    // JSONB를 String으로 처리 - JSONB 처리의 번거로움
    implements AttributeConverter<JsonNode, String> {

    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(JsonNode node) {
        if (node == null) return null;
        return node.toString();
    }
    @Override
    public JsonNode convertToEntityAttribute(String json) {
        try {
            return mapper.readTree(json);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("JSON 파싱 실패: " + json, e);
        }
    }
}
```

``` java
// @>, ->>, #>> 등 JSONB 연산자 직접 지원 X - 커스텀 함수 등록, 네이티브 쿼리 등으로 우회해야 해서 불편함
@Query(value = """
    SELECT * FROM posts
    WHERE content @> :filter::jsonb
    """, nativeQuery = true)
List<Post> findByContentContaining(@Param("filter") String filter);

// 호출
repo.findByContentContaining("{\"author\": \"kim\"}");
```

<br>

## 최종적 DB 접근 구조: JPA/Hibernate와 jOOQ의 병행

* ORM 기반 영속성 프레임워크와 jOOQ는 문제의 성격에 따라 장단점이 명확함
* 기능의 목적에 따라 장점을 적용할 수 있는 구조가 합리적이라고 판단함

### JPA/Hibernate

* ORM 기반 영속성 관리 및 dirty checking 이점 -> 데이터 정합성이 필수인 CUD 작업에 활용
* 완전한 엔티티를 구성해도 부담이 적은 일부 단순한 Read 작업에 활용

### jOOQ

* 영속성 맥락이 없어 필요한 데이터만 조회 가능 이점 -> 방대한 Read 작업 및 집계 쿼리에 활용
* 조건부 조회 작업에 적합한 동적 쿼리 기능 -> 다양한 조건 기반 게시글 검색에 활용

<br>

## 성과

* 쿼리 수 변화: JPA의 연관관계 탐색 과정에서 N+1이 발생하던 조회 쿼리 -> 명시적인 JOIN 쿼리로 재작성하여 쿼리 수 1회로 감소
* 쿼리 표현력: jOOQ가 JSONB를 직접 지원 -> JSONB와 JsonNode 사이를 용이하게 변환, ORM 환경에서 필요한 커스텀 타입 처리 및 매핑 로직 제거

<br>

****
