---
layout: single
title: 트러블슈팅 - 전역적인 예외 구조를 정립하며
---

# 📌 한 줄 요약

도메인 경계와 맞지 않던 전역 ErrorCode 구조를 분리하고, 
중복 예외를 통합하며, 런타임 메시지를 지원하는 예외 체계를 설계함. <br>

* ErrorCode 100+ → 도메인별 분리
* 중복 예외 클래스 32% 감소
* DynamicErrorCode로 런타임 메시지 지원

<br>

****

# 🧐 문제: 유지보수 어려운 예외 구조 + 늘어나는 중복 커스텀 예외 클래스

### 기존의 예외 처리 구조
**커스텀 예외 처리의 흐름** <br>
* 도메인 별 비즈니스 로직 수행 시 예외 발생
* 커스텀 예외 throw함 + 단일한 ErrorCode가 예외 상수 제공
* 전역적인 예외 핸들러(@RestControllerAdvice 기반)에서 클라이언트에게 에러 응답 발송 <br>

**프로젝트가 단순하여 Layered Architecture를 사용했을 때는 문제 X** <br>
**-> DDD + Clean Architecture 도입 후 도메인들이 나뉘며 문제 발생**

<br>

**단일한 ErrorCode에 모든 도메인이 의존**
* 예외 상수를 돌려쓰다보니 일반적인 응답 + 도메인에 종속된 응답이 혼재됨
* ErrorCode Enum의 비대화
    * Nickname 형식이 유효하지 않네. -> (ErrorCode 탐색 후) 관련 상수가 없네. ErrorCode에 추가해야겠다. -> (며칠 후) ...ErrorCode 하단에 이미 있었네? <br>

``` java
// 모든 도메인에서 공유하는 ErrorCode
public enum ErrorCode implements ResponseCode {

    // General case: 도메인에 종속되지 않은 일반적 응답
    GENERIC_ERROR(HttpStatus.INTERNAL_SERVER_ERROR.value(), "internal_server_error", "서버에 문제가 발생했습니다"),
    MISMATCH_INPUT_TYPE(HttpStatus.BAD_REQUEST.value(), "mismatch_input_type", "입력값의 타입이 올바르지 않습니다"),

    // domain: 도메인에 한정된 응답
    NOT_FOUND_COMMENT_LIKE(HttpStatus.NOT_FOUND.value(), "not_found_comment_like", "댓글의 좋아요 값이 존재하지 않습니다"),
    // 게시글 & 댓글 도메인에 국한된 ErrorCode 상수. 그러나 모든 도메인이 공유함
    NOT_FOUND_AUTHOR(HttpStatus.NOT_FOUND.value(), "not_found_author", "게시글 작성자가 존재하지 않습니다"),
    
    ...
    // 상단의 NOT_FOUND_COMMENT_LIKE와 중복
    COMMENT_LIKE_NOT_FOUND(HttpStatus.NOT_FOUND.value(), "comment_like_not_found", "댓글의 좋아요 값이 없습니다"),

    // ... 100개 이상의 ErrorCode 상수

    private final int httpStatus;
    private final String code;
    private final String message;
}
```

<br>

**중복되는 커스텀 예외 클래스의 증식**
* DDD에 따라 Bounded Context 유지 -> 도메인들이 예외를 거의 공유하지 않음
* 도메인마다 "비슷한 이름의 구조가 동일한" 커스텀 예외 클래스가 과도하게 생성됨
* 커스텀 예외의 구조가 변경될 시 수정할 클래스가 많아짐
* 중복 예외 클래스들이 사용하는 ErrorCode 값만 다르고 하는 역할은 동일 <br>
 
``` java
// member 패키지의 커스텀 예외
EmptyValueException
InvalidEmailException
MemberExistsException

// post 패키지의 커스텀 예외
EmptyNicknameException // EmptyValueException의 Nickname 버전.
InvalidEmailException // member의 커스텀 예외와 일치. Bounded Context를 이유로 나눴을 뿐.
ExistsMemberException // MemberExistsException의 이름만 바꾼 형태
```

<br>

**런타임에 따른 동적 예외 메시지 전달 불가**
* 예외 응답 발생 시 Enum 자료형인 ErrorCode에만 의존
* Enum은 컴파일 타임에 구조가 고정됨 -> 동적으로 메시지 변경 불가능 <br>

``` java
public record NormalSignUpRequest(
    @Schema(
        description = "이메일",
        pattern = Regex.REGEX_EMAIL,
        example = "flowers32@gmail.com"
    )
    @NotBlank(message = "이메일이 비어 있습니다.") // 응답에 반영 불가능
    @Pattern(regexp = Regex.REGEX_EMAIL,
            message = "이메일 서식이 올바르지 않습니다.") // 응답에 반영 불가능
    String email,
```

<br>

**결론**
* **불명확한 예외 구조 -> 개발자의 인지 비용 증가**
* **중복되는 커스텀 예외 클래스, ErrorCode 상수 -> 예외 체계의 관리 어려움**
* **런타임의 결과를 반영한 예외 응답 X -> 클라이언트가 상황에 대한 명확한 인지 불가**
* **= 지속 불가능한 예외 구조**

<br>

****

# ✨ 해결 및 성과: 커스텀 예외 구조 정립 + 기존의 예외 구조 리팩토링

### 🔍 단일한 ErrorCode 분리 + 명명 규칙 정립
**적용 범위에 따른 ErrorCode 분할**
* 특정 도메인의 맥락이 있는 예외 응답: 도메인 전용 ErrorCode (PostErrorCode, CommentErrorCode...)
* 도메인 맥락 없는 일반적인 예외 응답: GeneralErrorCode
* 특정 프레임워크와 관련된 예외에 대한 예외 응답: 프레임워크 전용 ErrorCode (SecurityErrorCode, EntityErrorCode...)

**ErrorCode의 명명 규칙 정립**
* 예외 발생 조건 + 발생 대상
* ✅ NOT_FOUND_POST | ❌ POST_NOT_FOUND <br>

``` java
// PostErrorCode: 멀티파트 데이터를 처리하므로 관련 에러 상수 보관
UNSUPPORTED_FILE(HttpStatus.FORBIDDEN.value(), "unsupported_file", "지원되지 않는 파일 타입입니다"),
FILE_LIMIT_EXCEEDED(HttpStatus.BAD_REQUEST.value(),"file_limit_exceeded","파일 개수 또는 크기 제한을 초과했습니다")

// GeneralErrorCode: 커스텀 예외가 아닌 예외 발생 시 사용
// 예외 예시: HttpMessageNotWritableException, ObjectOptimisticLockingFailureException
GENERIC_ERROR(HttpStatus.INTERNAL_SERVER_ERROR.value(), "internal_server_error", "서버에 문제가 발생했습니다"),
MISMATCH_INPUT_TYPE(HttpStatus.BAD_REQUEST.value(), "mismatch_input_type", "입력값의 타입이 올바르지 않습니다")

// SecurityErrorCode: Spring Security 전용 커스텀 예외에 사용
BANNED(HttpStatus.UNAUTHORIZED.value(), "banned", "밴 처리 된 계정입니다"),
DELETED(HttpStatus.UNAUTHORIZED.value(), "deleted", "삭제된 계정입니다")
```

<br>

### 🔍 중복되는 커스텀 예외 통폐합 + 명명 규칙 정립
**예외 발생 조건이 달라야 커스텀 예외 생성 가능**

``` java
✅ InvalidValueException, EmptyValueException

❌ InvalidMemberIdException, InvalidEmailException
-> InvalidValueException으로 통합 + valueName 필드로 원인 명시
```

**커스텀 예외의 명명 규칙 정립**
* 예외 발생 조건 + 대상 + Exception
* ✅ NotFound + Entity + Exception
* ❌ Credential + Invalid + Exception <br>

<br>

### 🔍 런타임 맥락을 위한 DynamicErrorCode 클래스 추가
* 런타임 맥락에 따라 메시지를 수정하는 클래스 추가 <br>

``` java
@Getter
@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
public class DynamicErrorCode implements ErrorCode{

    private final int httpStatus;
    private final String code;
    private final String message;

    // 메시지 변경만 가능 - 일관된 예외 응답 보장
    public static DynamicErrorCode create(ErrorCode errorCode, String message) {
        return new DynamicErrorCode(errorCode.getHttpStatus(), errorCode.getCode(), message);
    }
}
```

``` java
// 용례: MethodArgumentNotValidException처리 시 사용
String message = ex.getBindingResult().getAllErrors().getFirst().getDefaultMessage();

DynamicErrorCode dynamicErrorCode = DynamicErrorCode.create(GeneralErrorCode.INVALID_INPUT, message);
```

<br>

### 📍 수치로 측정한 성과
* **삭제된 커스텀 예외 클래스 수량:** 31개 
* **현재의 커스텀 예외 클래스 수량** 66개
* **결론:** 중복 예외 클래스 약 32% 단축

<br>

### 📍 예외 구조 재정립 + 단축된 커스텀 예외 클래스의 의미는?
**코드 재사용성 증가** <br>
* (기존) 도메인마다 중복되는 예외 클래스 및 전역으로 공유되는 ErrorCode 상수 생성
* (현재) 전역적인 예외 클래스와 ErrorCode 재사용 + 도메인 맥락 전달 필요 시 도메인마다 개별적으로 예외 클래스와 ErrorCode를 구성 <br>

**개발자의 인지 부담 경감**
* (기존) 비일관적 명명 체계로 인한 탐색 과정 장기화 + 중복 예외 클래스의 증식으로 유지보수 어려움
* (현재) 일관된 명명 규칙에 따른 용이한 탐색 + 모호한 예외 구조로 발생했던 의사결정 시간 단축
* 만들어진 예외 클래스를 어디에 두지? -> 도메인 전용 예외니까 도메인 패키지 내부에 두자.

**결론: 단순한 코드 감축 이상의 의미**
* **재사용성:** 전역적으로 예외 클래스 공유 + 중복 예외 클래스 생성 방지
* **생산성:** 예외 처리를 위한 개발자의 인지 부하 최소화
* -> **지속 가능한 예외 구조**

<br>

****

# 🌿 회고 1: 구조는 스스로를 말할 수 있어야 한다.
전역적인 예외 구조를 정립하며 가장 치열하게 고민한 바는 다음과 같다.

<br>

***팀원들이 이해하기 쉬우면서도, 안정적이고 지속 가능한 예외 구조란?***

<br>

나의 팀원들, 그리고 예외 구조를 정립한 후의 미래의 나는 **비즈니스 로직**에 집중해야 한다. 비즈니스 로직이 조명을 받는 배우라면, 예외 처리는 배우가 올라 서는 무대이다. <br>

물론 Confluence에 예외 구조에 대한 문서를 작성해놓았다. 그러나 개발자는 Confluence보다 IDE를 더 오랜 기간 동안 보는 사람이다. <br>
*구조는 스스로를 말할 수 있어야 한다.*

<br>

개발자 팀원이 예외 구조를 봤을 때 아래를 파악할 수 있어야 한다.<br>
* 도메인들이 공유하는 예외가 있네? -> 중복 예외 클래스 생성 금지. <br>
* 범위마다 ErrorCode가 따로 있네? -> 하나의 GeneralErrorCode에 상수 몰아넣지 않기. <br>
* 런타임 맥락을 반영하는 DynamicErrorCode는 메시지밖에 수정 못 하네? -> 구조적으로 제약을 걸어 수정하면 안 되는 데이터에 대한 접근성 차단.

<br>

내가 생각하는 좋은 예외 구조란 <br>
**구조 자체를 통해 의도가 읽혀서 이해하기 쉽고** <br>
**신뢰할 수 있어서 비즈니스 로직을 개발하는 팀원이 마음 놓고 예외를 던질 수 있는** <br>
구조이다.

<br>

****

# 🌿 회고 2: 여전히 남은 생각거리

*내가 만든 예외 구조는 완벽하지 않다.* 여전히 고민할 사안들이 있다.

<br>

### Logging 및 모니터링 최적화
예외는 클라이언트에게 예외 응답을 보내는 수단이기도 하지만, 서버 내부의 로깅 및 디버깅 용도의 의미도 크다. <br>
프로젝트 내부에 Logging 계층이 있었고, 예상대로 예외의 message 필드를 사용하고 있었다. 이 때 들었던 생각은 다음과 같다. <br>
* 나는 예외 구조의 단순명료함과 지속 가능성에 초점을 맞췄다.
* 다음으로 고려할 것은 프로젝트 운영에 필요한 정보로서의 예외다.
* 그러한 예외는 중요도(Severity), 환경(env), 발생 시각과 같은 정보를 포함한다.

<br>

지금의 커스텀 예외 필드에는 toString() 메서드가 있어서, Logging 계층에 예외 클래스의 필드에 대한 정보를 반영할 수 있다. <br>
그러나 프로그램 운영의 자료로 쓰기에는 부족한 느낌이 든다.
* 예외를 구성하며 로그 레벨 전략을 정하는 방법이 무엇인지
* 커스텀 예외 Logging에 있어 우선적으로 추가할 필드의 종류가 무엇인지 <br>

이런 사안들에 대한 추가적인 고민 및 개선이 필요하다. 추후에 예외에 Logging 및 모니터링에 필요한 필드를 추가하여 실효성을 검증해볼 계획이다.

<br>

### 스스로를 말할 수 있는 구조의 실현 가능성?
내가 "스스로 말할 수 있는 예외 구조"를 구성했다고는 하지만, 예외 구조는 전체 애플리케이션에 걸쳐 구성된 방대한 범위를 포괄한다. 즉 직관적으로 이해하기 어려운 측면이 있다. <br>
* 팀원들에게 Confluence에 "전역적 예외 구조" 문서가 신설되었음을 공지했다.
* 영향 범위가 넓은 만큼, 짧은 비대면 회의를 통해 문서의 내용이 실제 코드에 반영되어 있는 모습을 시연했다.

<br>

하지만 여전히 "보기만 해도 알 수 있는 직관적인 구조"를 실체화하기는 어렵다. <br>
* 팀원이 Config 클래스 관련 커스텀 예외를 추가하며, "Config 클래스가 애플리케이션 전체에 영향을 미치니까 ConfigException을 전역적인 global 패키지에 넣자"고 하였다.
    * 커스텀 예외 클래스의 위치를 선정하는 근거는 예외가 종속되는 대상이지, 종속의 대상(Config 클래스)의 영향 범위와는 무관하다.
    * global > exception이 아닌 config > exception에 위치해야 함을 알렸다.
* 다른 팀원이 게시글 도메인에 MultipartDataProcessingException을 생성했다.
    * 예외는 발생 원인이 잘못된 대상보다 중요하다. 해당 예외의 이름에는 "무엇이 잘못되었다"만 내포되어 있다.
    * 예외의 원인을 명확히 밝혀달라고 부탁했다. (InvalidMultipartDataFormatException 등)

<br>

이는 *팀원의 역량 보다는 구조적인 문제*, 즉 학습 곡선에 따른 문제이다. <br>

* 내가 예외 구조 정립의 담당자이므로, 당연히 예외에 대한 이해도가 내가 가장 높다.
* 다른 팀원이 개발한 기능을 마치 자기 것 처럼 이해하기는 어렵다. 일련의 판단 과정과 트러블슈팅을 경험하지 못했기 때문이다. (나 또한 다른 팀원이 개발한 기능에 대해 대략적으로만 알고 있다)
* 예외 구조는 유일한 정답이 없는 문제다. 우리 팀 만의 관례를 익히려면 시간이 필요할 것이다. <br>

결국 구조만으로 모든 의도를 전달하기는 어렵다. 꾸준한 리뷰와 소통이 필요하다. 팀의 예외 구조와 맞지 않는 점이 발견될 시, 코드 리뷰를 통해 보완해갈 예정이다.
