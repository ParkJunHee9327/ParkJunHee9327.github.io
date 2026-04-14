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
