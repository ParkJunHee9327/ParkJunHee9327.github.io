---
layout: single
title: 트러블슈팅 - PostgreSQL의 EXPLAIN으로 이룬 댓글 조회 API의 DB 쿼리 속도 단축
---

# 🧐 문제: 댓글 조회 API 실행에 필요한 인덱스 최적화 부족

## 문제 상황: 댓글 조회 API의 최적화 인덱스 부재
* 해당 API는 특정 게시글을 화면에 표시할 시 하단에 배치할 댓글을 표시하는 조회 API임
* 댓글 정보 뿐 아니라 댓글 작성자및 댓글 좋아요의 정보를 동반하는 **다중 JOIN 구조**
* **게시글 식별자에 해당하는 댓글** 기준 **created_at 기반 최신순 정렬**을 하는 작업이 포함됨
* -> 테이블 분석 후 *일부 인덱스가 누락* 되었음을 식별함

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
    -- 특정 게시글에 해당된 댓글 식별을 위한 인덱스 부재
    -- 최신순 정렬을 위한 인덱스 부재
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
* 경험적으로 커뮤니티에서 소수의 헤비 유저가 활동을 주도하는 사례를 다수 목격함
* 특정 사용자에게 댓글 데이터가 집중되는 상황에서 성능 문제가 증폭될 가능성이 있다고 판단
* **방향성 1:** 데이터 집중도를 극단적으로 높인 Edge Case를 구성하여 인덱스 유무에 따른 성능 차이 분석
* **방향성 2:** DB 쿼리 성능이 주요 측정 대상이므로 API 응답에 수반되는 DTO 매핑 등 작업은 배제하고 측정

<br>

****

# ✨ 가설 검증 및 테스트 결과

## 테스트 환경
* **측정 도구**
    * PostgreSQL의 EXPLAIN 사용
    * JSON 파싱 등 응답에 수반되는 측정 대상이 아닌 작업 제외
* **테스트 데이터**
    * 한 게시글에 댓글 10,000개 집중
    * 댓글 작성자는 3명으로 편중
    * -> 소수 유저의 조회가 반복되는 구조
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
* **도입한 인덱스**

```
-- 특정 게시글에 등록된 댓글 식별을 위함
CREATE INDEX IDX_COMM_COMMENT_POST_ULID ON comm_comment(post_ulid);
-- 게시글 식별자로 선별된 댓글을 최신순 정렬하기 위함
CREATE INDEX IDX_COMM_COMMENT_POST_SORT ON comm_comment(post_ulid, created_at);
```

* **주요 병목 지점 개선:** nested loop의 수행 시간이 0.004ms -> 0.002ms로 약 절반 가량 단축
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
