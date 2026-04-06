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

### 실용성과 Bounded Context 사이의 균형

도메인들끼리 어디까지 공유해야 하는지, 어디까지 도메인 전용으로 구분할지의 고민이 있다.
* 도메인들이 공유하는 VO를 kernel로서 관리할 때, 시간이 흘러도 공유하게 될까?
* 애플리케이션의 서버가 1대이고, 도메인들이 동일한 DB에 접근한다. 동일한 JPA 엔티티를 공유해도 되지 않을까?

<br>

Bounded Context를 유지하려고 나누는 것 vs 전역적으로 공유하여 관리할 클래스의 수를 줄이는 것 사이의 고민이 있다. 지금의 결론은 이러하다.
* 특별한 행동 없이 값 보관 자체에 의의가 있으며, 애플리케이션 내에서 동일한 검증 로직을 적용해야 하는 VO를 kernel로서 공유한다.
``` java
// 현재 kernel로서 공유되고 있는 Nickname VO
public class Nickname {
    private final String value;

    public static Nickname create(String value) {
        if (value == null || value.isBlank()) {
            throw new EmptyValueException(KernelErrorCode.EMPTY_NICKNAME, "nickname");
            // 도메인 마다 동일한 정규표현식 사용
        } else if (!PATTERN_NICKNAME.matcher(value).matches()) {
            throw new InvalidValueException(KernelErrorCode.INVALID_NICKNAME_FORMAT, "nickname");
        }
        return new Nickname(value);
    }
```
* 유지보수의 편의성을 위해 JPA 엔티리를 도메인들 끼리 공유하되, 도메인 내에서 엔티티를 도메인 맞춤 VO로 치환하여 Bounded Context를 유지한다.
```
                        // 여기서 JPA 엔티티가 도메인 맞춤 VO로 매핑됨
                                           |
                                           v
identity의 Bounded Context 내부 -  jOOQ 리포지토리(identity - framework 계층) - 전역으로 공유되는 JPA 엔티티 
```

<br>

### 비즈니스 로직에 대한 DDD의 Rich Domain Model과 Clean Architecture의 충돌
* DDD의 Rich Domain Model에 따르면 도메인 클래스가 상태 + 행동(비즈니스 로직)을 책임진다. <br>
* Clean Architecture -> 비즈니스 로직을 use case에서 관리한다. <br>

*"비즈니스 로직을 어디에 둘 것인가"* 의 쟁점이 있다.

<br>

지금의 결론은 다음과 같다.
* DDD는 도메인 클래스의 관리 전략이지, 애플리케이션 전체 아키텍처를 정의하지 않음
* Clean Architecture는 애플리케이션 전체 아키텍처를 정의함
* Rich Domain Model 채택 시, 도메인 클래스 내에 행동이 모두 들어가게 되므로 -> 기능에 따라 use case로 비즈니스 로직을 관리하는 Clean Architecture와 부자연스럽게 통합되리라 예상됨 <br>

-> 따라서, Rich Domain Model이 아닌 Clean Architecture에 따라 비즈니스 로직을 관리함 <br>
-> 단, 앞서 언급했듯 현재 비즈니스 로직이 단순하기 때문에 adapter 계층에서 오케스트레이션 수행 <br>
-> 비즈니스 로직의 복잡도가 늘어날 시, adapter -> use case 계층으로 옮길 예정

<br>

사실, 여전히 "use case 계층에서 비즈니스 로직을 관리하면 결국 Anemic Domain Model로 회귀하는 거 아닌가?"하는 의문이 있다. <br>
Rich Domain Model의 요점은 "도메인에 대한 상태 뿐 아니라 행동도 통합적으로 캡슐화하는 것"이라고 생각한다. 비록 도메인 클래스 내부에 비즈니스 로직이 위치해있지는 않으나, 도메인을 패키지별로 나눠 명확한 Bounded Context를 설정했으니 나름의 해결책이라 하겠다.

<br>

### 여전히 남은 문제, 인프라와 전역적인 클래스들
인프라 및 Cross-cutting concern은 DDD도, Clean Architecture도 "이렇게 하라"는 지침이 없다.
* Config나 logging 클래스들을 어디에 배치하나? 관련된 유틸리티 클래스는 어디에 두나?
* 특정 프레임워크 전용 검증 및 유틸리티 클래스는 어디에 둬야 할까?
* 전역적으로 공유했다가 나중에 의존성이 복잡해지지 않을까?

<br>

지금의 결론은 다음과 같다.
* Cross-cutting concern이다 -> infrastructure 패키지
* 어떠한 도메인과도 무관하며, 프레임워크에 대한 의존성이 강하다 -> 전역 framework 패키지
* 순수한 Java라서 어느 도메인이든 공유할 수 있다 -> shared 패키지


<br>

****
