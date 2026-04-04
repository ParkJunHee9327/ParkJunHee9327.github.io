---
layout: single
title: 트러블슈팅 - DDD + Clean Architecture로 리팩토링한 이유
---

# 🧐 문제: 유지보수 어려운 아키텍처로 인한 개발 기간 연장

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

# ✨ 해결 및 성과: DDD(Domain-Driven Design) + Clean Architecture 도입

### 🔍 도입 이유 및 성과
* **DDD**
    * **도입 이유:** 도메인들의 경계가 불명확함, 도메인 객체가 단순한 Data Holder 역할을 함
    * 비즈니스 로직 중 비중이 높던 ValidationService -> 도메인 클래스에 흡수 (검증 로직)
    * 도메인 간 명확한 경계 설정, 클래스 수정으로 인한 잠재적 파급력 감소
* **Clean Architecture**
    * **도입 이유:** DDD와 통합 가능 + 도메인과 프레임워크를 분리하는 아키텍처 필요
    * 도메인/프레임워크 간 명확한 분리
    * 클래스 추가 시, 어디에 추가해야 하는지에 대한 혼란 감소
    ```
    📂 [domain_name]
    ├─📂 adapter    # Interface Adapter: 외부 시스템 연결, 데이터 매핑 및 오케스트레이션
    ├─📂 domain     # Enterprise Business Rules: 핵심 비즈니스 로직 (Aggregate, Entity, VO)
    ├─📂 framework  # Frameworks & Drivers: 기술 구현체 (Service, RestController, Persistence)
    └─📂 usecase    # Application Business Rules: 앱 사양 정의 (Port, DTO, Model)
    ```

<br>

### 🔍 애플리케이션의 맥락을 고려한 차별성
* **도메인 클래스에 Lombok 허용**
    * Lombok은 장황한 메서드를 어노테이션으로 축약해줌 -> Lombok이 도메인 클래스의 핵심 기능에 미치는 영향력이 미미함 + 깔끔한 코드로 인한 가독성 증대 이점 -> 허용
* **오케스트레이션을 수행하는 Adapter 계층**
    * 오케스트레이션은 use case 계층이 수행하는 게 일반적
    * 그러나 애플리케이션의 비즈니스 로직 대부분이 단순한 검증 및 DB 쿼리 수행 위주임
    * 개별적인 use case 클래스 -> 단순한 비즈니스 로직에 있어 overkill임
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

### 📍 수치로 측정한 성과
* *외부 변수를 완벽하게 통제한 결과가 아니므로 참고 지표로 해석해야 함*
* **측정 대상:** 댓글 삽입 API, 댓글 수정 API. (동일 인물이 개발)
    * CRUD 기능이므로 두 API의 개발 난이도가 유사함.
    * 비즈니스 로직 난이도가 쉬움 -> 아키텍처의 구조가 생산성에 끼치는 영향력 극대화.
* **측정 범위:** API의 요구사항 분석 + 아키텍처 설계 + 테스트 코드 작성 후 테스트 완료.
* **측정 기준:** Jira의 백로그로 등록된 기능 개발 기간.
* **측정 결과:** 기능 개발 기간 약 30% 단축

<br>

### ✅ 생산성 비교 표

| 비교 항목 | 리팩토링 전(댓글 삽입) | 리팩토링 후(댓글 수정) | 비고 |
|:---|:---|:---|:---:|
| 아키텍처 | Anemic Domain Model <br> + Layered Architecture | DDD <br> + Clean Architecture | - |
| 비즈니스 개발 난이도 | 쉬움 | 쉬움 | 변수 통제 |
| 아키텍처 이해 난이도 | 어려움 | 쉬움 | 측정 대상 |
| 개발 소요 시간 | 약 7일 | 4~5일 | 약 30% 감소 |

<br>

****

# 🌿 회고 1: 혹자의 질문. DDD + Clean Architecture로 바꾸면 구조가 너무 비대하지 않나?

아키텍처 리팩토링을 완료한 뒤 계층이 더 다층적으로 변했다. 누군가는 이걸 보고 "구조가 더 복잡해졌네"라고 할 지 모른다. <br>
내가 절대적으로 옳은 건 아니다. 다만, 나의 경험에 의하면 "다층적인 구조 = 더 복잡한 구조"가 아니라 "다층적인 구조 = 책임 간 명확한 분리 = 이해하기 쉬운 구조"였다.

<br>

기존의 계층에는 Service 계층에 시스템 명세(인터페이스)와 비즈니스 로직 구현체(클래스)가 함께 있었다. 클래스를 하나만 추가해도 파급력을 파악하기 어려웠다.
* '누가 누구에게 의존하고 있는가? 내가 이 Service를 추가하면 순환 의존성이 발생하는가?'
* 'Service의 이 메서드를 바꾸면 어디까지 영향이 가는가?' <br>

눈에 보이지 않는 잠재적인 영향을 예견하려 애쓰는 과정, 이게 개발 기간을 늘리는 주요한 원인이었다.

<br>

지금은 무엇이 어디에 들어갈 지 확실하다.
* '도메인 별 공통적인 VO는 Kernel로서 공유한다'
* '도메인에 한정된 클래스는 공유하지 않는다'
* '시스템 명세는 use case에, 구현체는 framework에 있다' <br>

추가할 기능이 생기면 다음 과정을 거친다. <br>

"마이페이지에서 사용자의 인증 정보를 받고 싶다고?" -> "그럼 일반/소셜에 국한되지 않는 identity 도메인 내에서 개발해야겠다." -> "도메인에 특화된 VO는 identity 내에서 만들고, Nickname VO는 Kernel에서 공유하니까 가져다 쓰자."

<br>

**개발자가 온전히 비즈니스 로직에 집중할 수 있는 아키텍처. 추가되는 기능이 다른 도메인에 영향을 미치지 않을 거라고 안심할 수 있는 아키텍처.** <br>
이게 내가 생각하는 DDD + Clean Architecture의 가장 큰 이점이다.

<br>

****

# 🌿 회고 2: 그래서... DDD + Clean Architecture가 최고라는 거야?

결론부터 말하자면, 물론 *아니다*. <br>
아키텍처를 리팩토링하며 아쉬웠거나 더 고민해야 하는 지점들이 있었다.

<br>

**실용성과 Bounded Context 사이의 균형**
* 어디까지 공유할 것인가? 공유하는 VO를 Kernel로 뺐을 때 시간이 흘러도 공유할 수 있을까라는 의구심, 애플리케이션의 서버가 1대고 도메인들이 동일한 DB에 접근하니까 동일한 JPA 엔티티를 공유하자는 생각...
* Bounded Context를 위해 나누는 것 vs 전역적으로 공유해서 관리 대상인 클래스를 줄이는 것 사이의 균형에 대한 고민

<br>

**DDD의 Rich Domain Model과 엇맞는 Clean Architecture**
* DDD의 Rich Domain Model -> 도메인 클래스가 상태 + 행동까지 책임짐
* Clean Architecture -> 비즈니스 로직(행동)을 use case에서 책임짐
* 결국 Rich Domain Model의 "도메인은 행동도 책임진다"를 이행하지 못 함. DDD는 도메인 클래스 관리 전략이고, Clean Architecture는 애플리케이션 전체를 구성하는 아키텍처이기 때문에 Clean Architecture의 손을 들어줌. DDD는 "애플리케이션 전체 아키텍처는 이러이러하게 하라"고 알려주지 않음.
* 이로 인해 발생하는 영향이 뭐가 있을까?

<br>

**여전히 남은 문제, 인프라와 전역적인 클래스들**
* Config나 logging 전용 클래스들을 어떻게 관리하나? 관련된 유틸리티 클래스는 어디에 두나?
* 횡단 관심사인 Spring Secuirty(보안) + 전역적인 예외 구조 + Config 클래스가 있는 패키지 + Logging 패키지가 모두 global 패키지 내부에 있음.
* 인프라와 전역적인 클래스를 다루는 구조는 DDD도, Clean Architecture도 알려주지 않음.

<br>

****
