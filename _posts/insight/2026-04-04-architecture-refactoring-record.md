---
layout: single
title: 트러블슈팅 - DDD + Clean Architecture로 리팩토링
---

# 문제: 유지보수 어려운 아키텍처로 인한 개발 기간 연장

* 도메인이 단순한 Data Holder 역할을 함
    * 상태 관리 X, 행동 X
    * "도메인 객체의 존재 의미가 무엇이며, 언제 사용해야 하는가"의 혼란
* 도메인들 사이의 경계가 모호함
    * 여러 도메인들이 공통된 클래스에 의존
    * 클래스 수정 시 다른 도메인에 일으킬 파급력에 대한 불안감 유발
* 비즈니스 로직을 다루는 클래스를 모두 Service로 명명 + Service들이 서로 의존함 상황
    * 새로운 Service 추가 시 순환 의존성 등 잠재적 문제 가능성 도출
* **결론: Anemic Domain Model + Layered Architecture의 Service 계층 비대화 -> 유지보수 어려운 아키텍처 형성 -> 기능 개발 기간 증가**

<br>

****

# 해결 및 성과: DDD(Domain-Driven Design) + Clean Architecture 도입

### 도입 이유 및 성과
* **DDD**
    * **도입 이유:** 도메인들의 경계가 불명확함, 도메인 객체가 단순한 Data Holder 역할을 함
    * 비즈니스 로직 중 비중이 높던 ValidationService -> 도메인 클래스에 흡수 (검증 로직)
    * 도메인 간 명확한 경계 설정, 클래스 수정으로 인한 잠재적 파급력 제거
* **Clean Architecture**
    * **도입 이유:** DDD와 통합 가능 + 도메인과 프레임워크를 분리하는 아키텍처 필요
    * 도메인/프레임워크 간 명확한 분리
    * 클래스 추가 시, 어디에 추가해야 하는지에 대한 혼란 제거
    ```
    📂 [domain_name]
    ├─📂 adapter    # Interface Adapter: 외부 시스템 연결, 데이터 매핑 및 오케스트레이션
    ├─📂 domain     # Enterprise Business Rules: 핵심 비즈니스 로직 (Aggregate, Entity, VO)
    ├─📂 framework  # Frameworks & Drivers: 기술 구현체 (Service, RestController, Persistence)
    └─📂 usecase    # Application Business Rules: 앱 사양 정의 (Port, DTO, Model)
    ```

<br>

### 애플리케이션의 맥락을 고려한 차별성
* **도메인 클래스에 Lombok 허용**
    * Lombok은 장황한 메서드를 어노테이션으로 축약해줌 -> Lombok이 도메인 클래스의 핵심 기능에 미치는 영향력이 미미함 + 깔끔한 코드로 인한 가독성 증대 이점 -> 허용
* **오케스트레이션을 수행하는 Adapter 계층**
    * 오케스트레이션은 use case 계층이 수행하는 게 일반적
    * 그러나 애플리케이션의 대부분의 비즈니스 로직이 단순한 검증 및 DB 쿼리 수행 위주임
    * 따라서 adapter에서 오케스트레이션 수행 + use case는 시스템 사양(인터페이스) 정의 및 model 보관에 집중
    ``` java
    // Validate(Domain) -> Read(Repository) -> Update(Repository)
    public void modifyEmail(UUID memberUuid, EmailModificationRequest request) {
        if(!readRepository.existsByEmail(Email.create(request.currentEmail()))) {
            throw new NotFoundEntityException(EntityErrorCode.NOT_FOUND_MEMBER, TableName.SITE_MEMBER_AUTH);
        } else {
            updateRepository.updateEmail(AccountId.create(memberUuid), Email.create(request.newEmail()));
        }
    }
    ```

<br>

### 수치로 측정한 성과
* *외부 변수를 완벽하게 통제한 결과가 아니므로 참고 지표로 해석해야 함*
* **측정 대상:** 댓글 삽입 API, 댓글 수정 API. (동일 인물이 개발)
    * CRUD 기능이므로 두 API의 개발 난이도가 유사함.
    * 비즈니스 로직 난이도가 쉬움 -> 아키텍처의 구조가 생산성에 끼치는 영향력 극대화.
* **측정 범위:** API의 요구사항 분석 + 아키텍처 설계 + 테스트 코드 작성 후 테스트 완료.
* **측정 기준:** Jira의 백로그로 등록된 기능 개발 기간.
* **측정 결과:** 기능 개발 기간 약 30% 단축

<br>

### 성능 비교 표
| 비교 항목  | 리팩토링 전(댓글 삽입) | 리팩토링 후(댓글 수정) | 비고 |
|:-------------:|:-------------:|:-------------:|:-------------:|
| 아키텍처 | Anemic Domain Model <br> + Layered Architecture | DDD + <br> Clean Architecture | - |
| 비즈니스 개발 난이도 | 쉬움 | 쉬움 | 변수 통제 |
| 아키텍처 이해 난이도 | 어려움 | 쉬움 | 측정 대상 |
| 개발 소요 시간 | 약 7일 | 4~5일 | 약 30% 감소 |

<br>

****

# 회고: Anemic Domain Model + Layered Architecture
