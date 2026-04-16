---
layout: single
title: 트러블슈팅 - PostgreSQL의 EXPLAIN으로 이룬 댓글 조회 API의 DB 쿼리 속도 단축
---

# 🧐 문제: 댓글 조회 API 실행에 필요한 인덱스 최적화 부족

## 문제 상황: 댓글 조회 API의 최적화 인덱스 부재
* 해당 API는 특정 게시글을 화면에 표시할 시 하단에 배치할 댓글을 표시하는 조회 API임
* 댓글 정보 뿐 아니라 댓글 작성자및 댓글 좋아요의 정보를 동반하는 **다중 JOIN 구조**
* **게시글 식별자에 해당하는 댓글** 기준 **created_at 기반 최신순 정렬**을 하는 작업이 포함됨
* -> 댓글 테이블 구조 분석 후 쿼리 패턴 대비 적절한 인덱스가 구성되어 있지 않음을 확인

``` bash
                        "public.comm_comment" 테이블
     필드명     |            형태             | 정렬규칙 | NULL허용 | 초기값
----------------+-----------------------------+----------+----------+--------
 post_ulid      | character varying(26)       |          | not null |
 path           | text                        |          | not null |
 auth_memb_uuid | uuid                        |          |          |
 like_count     | integer                     |          | not null |
 content        | character varying(600)      |          | not null |
 is_deleted     | boolean                     |          | not null |
 created_at     | timestamp without time zone |          | not null |
 updated_at     | timestamp without time zone |          | not null |
인덱스들:
    -- 게시글 식별자를 바탕으로 최신순 정렬을 하기 위한 인덱스의 부재
    "PK_COMM_COMMENT" PRIMARY KEY, btree (post_ulid, path)
    "idx_comm_comment_content_trgm" gin (content gin_trgm_ops)
```

<br>

## 문제 해결 중요성: 애플리케이션의 맥락
* 문제가 되는 API는 "특정 게시글에 등록된 댓글 및 댓글 연관 정보를 조회하는" API임
* 해당 API가 개발된 사이드 프로젝트는 식물 커뮤니티로, 게시글 및 댓글 도메인의 비중이 큼
* 특히 게시글 및 게시글에 등록된 댓글을 조회하는 작업의 횟수가 많은 구조

<br>

## 가설 및 테스트 방향성
* 커뮤니티 서비스 특성 상 <u>특정 게시글에 댓글이 집중되는 상황</u>이 발생할 가능성 있음
* 이러한 상황에서 댓글 수 증가에 따라 정렬 비용이 주요 병목이 되고, JOIN 또한 부가적인 비용 증가 요인이 될 수 있다고 판단함
* 특히 댓글 조회 쿼리는 작성자 및 프로필 정보를 포함하기 위해 다중 JOIN을 수행하므로, 댓글 수 증가에 따른 전체 쿼리 비용 변화를 함께 관찰하고자 함
* **방향성 1:** 게시글에 댓글 데이터가 집중된 Edge Case를 구성하여, 인덱스 유무에 따른 탐색 및 정렬 비용 차이 비교
* **방향성 2:** DB 쿼리 성능이 주요 측정 대상이므로 API 응답에 수반되는 DTO 매핑 등은 제외
* **방향성 3:** 게시글 기준 필터링과 정렬을 동시에 만족하는 인덱스 설계 검증

<br>

****

# ✨ 가설 검증 및 테스트 결과

## 테스트 환경
* **측정 도구**
    * PostgreSQL의 EXPLAIN ANALYZE 사용
    * JSON 파싱 등 응답에 수반되는 측정 대상이 아닌 작업 제외
* **테스트 데이터**
    * 한 게시글에 댓글 10,000개 집중
    * 댓글 작성자는 3명으로 편중
* **측정 횟수**
    * 데이터 캐시 등 요소를 고려하여 처음 2회 실행값은 배제
    * 이후 10번 반복 측정

<br>

## 테스트 결과: 인덱스 도입 전/후
### 인덱스 도입 전
* **주요 병목 지점:** 회원 프로필과 JOIN 시 Seq Scan을 수행하여 전체 실행 시간의 **70% 이상**을 차지
* **부차적 병목 지점:** JOIN이 완료된 댓글 데이터를 정렬한 시간 자체는 미미함. (약 25ms) 그러나 데이터 양이 많아질수록 부담이 커질 수 있는 구간임.
* **결과:** 525ms, 278ms, 294ms, 374ms, 396ms, 267ms, 125ms, 123ms, 269ms, 172ms (평균 282.3ms)
```
-- 병목 구간 1
-> Nested Loop (cost=5.49..66.80 rows=1 width=1095) (actual time=1.871..237.516 rows=10000 loops=1)
-- 병목 구간 2
->  Bitmap Index Scan on "PK_COMM_COMMENT"  (cost=0.00..4.42 rows=18 width=0) (actual time=1.402..1.402 rows=10010 loops=1)
```

### 인덱스 도입 후
* **가설에 따라 도입한 인덱스**

```
-- 게시글의 WHERE절 최적화 용도
CREATE INDEX IDX_COMM_COMMENT_POST_ULID ON comm_comment(post_ulid);
-- 게시글로 선별된 댓글의 ORDER BY절 최적화 용도
CREATE INDEX IDX_COMM_COMMENT_POST_SORT ON comm_comment(post_ulid, created_at);
```

* **주요 병목 지점 개선:** nested loop의 수행 시간이 237ms -> 113ms로 감소
* **부차적 병목 지점 개선:** 정렬 시간이 277ms -> 135ms로 절반 가량 단축
* **결과:** 107ms, 129ms, 137ms, 184ms, 127ms, 117ms, 131ms, 227ms, 153ms, 126ms (평균 143.8ms)

```
-- 중첩 반복문의 수행 시간 단축
-> Nested Loop ... (actual time=1.871..237.516 rows=10000 loops=1)
-> Nested Loop ... (actual time=0.791..113.356 rows=10000 loops=1)


-- 정렬 시간 단축
-> Sort  (cost=66.82..66.83 rows=1 width=1096) (actual time=276.255..277.381 rows=10000 loops=1)
-> Sort  (cost=144.85..144.85 rows=1 width=1096) (actual time=134.578..135.906 rows=10000 loops=1)
```

<br>

****

# ✨ 성과 및 후속 계획

## 테스트 결과 해석
* 인덱스 추가가 주요 변화였으나, 단일/복합 인덱스가 동시에 적용되어
각 인덱스의 개별 기여도를 분리하지는 못함
* 복합 인덱스(post_ulid, created_at) 순으로 **정렬된 상태의 데이터가 제공되면서 정렬 비용이 크게 감소**한 것으로 보임
* 개선 전/후 총 실행 시간 평균이 **282ms -> 143ms (약 45%)** 단축

### Nested Loop
* post_ulid 조건을 만족하는 댓글을 인덱스를 통해 빠르게 탐색하면서
Heap 접근 범위가 줄어듦 -> Nested Loop 전체 수행 시간이 감소함

```
-- 개선 전: 특정 게시글에 속한 댓글 탐색 시 comment의 기본 키 인덱스 사용
-> ->  Bitmap Index Scan on "PK_COMM_COMMENT"
-> Bitmap Heap Scan on comm_comment → actual time=1.517..14.243

-- 개선 후: 특정 게시글에 속한 댓글 탐색 시 comment의 post_id에 
-> Bitmap Index Scan on "IDX_COMM_COMMENT_POST_SORT"  (cost=0.00..4.66 rows=50 width=0) (actual time=0.658..0.659 rows=10000 loops=1)
-> Bitmap Heap Scan on comm_comment → actual time=0.705..6.759
```

### 정렬
* 복합 인덱스를 통해 일부 정렬된 상태로 데이터가 제공되면서 처리 비용이 크게 감소한 것으로 해석됨

```
-- 개선 전
-> Sort  (cost=66.82..66.83 rows=1 width=1096) (actual time=276.255..277.381 rows=10000 loops=1)

-- 개선 후
-> Sort  (cost=144.85..144.85 rows=1 width=1096) (actual time=134.578..135.906 rows=10000 loops=1)
```

## 한계점 및 후속 계획

### 한계점
* WHERE절과 ORDER BY절을 각각 최적화하려 했던 가설과 달리 실제로는 post_ulid로 구성된 단일 인덱스는 사용되지 않음 -> 단일 인덱스 제거 예정
* 개선 전화 후 모두 테스트 데이터 삽입 후 ANALYZE 미실행으로 인해 실험 일관성은 있으나 신뢰도가 부족함을 인지 -> 후속 실험에서 ANALYZE 실행 예정
* Edge Case를 위한 댓글 데이터의 수량이 다소 부족하다고 판단 -> 댓글 데이터 개수 늘릴 예정

### 후속 테스트 계획
### 검증 목적
1. 단일 인덱스를 제거한 상태에서 복합 인덱스만으로 동일한 성능이 유지되는지 확인
2. ANALYZE 적용을 통해 통계 정보가 실행 계획에 미치는 영향 확인
3. 데이터 규모를 확장하여 정렬 비용 및 인덱스 효율이 어떻게 변화하는지 검증
