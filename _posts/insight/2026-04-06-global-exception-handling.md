---
layout: single
title: 트러블슈팅 - 전역적인 예외 구조를 정립하며 얻은 이점
---

# 🧐 문제: 유지보수 어려운 예외 구조 + 늘어나는 중복 커스텀 예외 클래스

### 기존의 예외 처리 구조
**커스텀 예외 처리의 흐름**
```
도메인 별 비즈니스 로직 수행 시 예외 발생
-> 커스텀 예외 throw함 + 공유하여 쓰이는 ErrorCode의 상수 사용
-> 전역적인 예외 핸들러(@RestControllerAdvice 기반)에서 클라이언트에게 에러 응답 발송
```

**단일한 ErrorCode에 모든 도메인이 의존**
* 예외 상수를 돌려쓰다보니 일반적인 응답 + 도메인에 종속된 응답이 혼재됨
* ErrorCode Enum의 비대화
    * Nickname 형식이 유효하지 않네. -> (ErrorCode 탐색 후) 관련 상수가 없네. ErrorCode에 추가해야겠다. -> (며칠 후) ...ErrorCode 하단에 이미 있었네?
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

**중복되는 커스텀 예외 클래스의 증식**
* DDD에 따라 Bounded Context 유지 -> 도메인들이 예외를 거의 공유하지 않음
* 도메인마다 "비슷한 이름의 구조가 동일한" 커스텀 예외 클래스가 과도하게 생성됨
* 커스텀 예외의 구조가 변경될 시 수정할 클래스가 많아짐
* 중복 예외 클래스들이 사용하는 ErrorCode 값만 다르고 하는 역할은 동일
```
// member 패키지의 커스텀 예외
EmptyValueException
InvalidEmailException
MemberExistsException

// post 패키지의 커스텀 예외
// EmptyValueException의 Nickname 버전. 사용하는 ErrorCode 값만 다르고 예외 구조는 동일.
EmptyNicknameException
// member의 커스텀 예외와 일치. Bounded Context를 이유로 중복하여 쓸 뿐.
InvalidEmailException
// MemberExistsException의 이름만 바꾼 형태
ExistsMemberException
```

**런타임에 따른 동적 예외 메시지 전달 불가**
* 예외 응답 발생 시 Enum 자료형인 ErrorCode에만 의존
* Enum은 컴파일 타임에 구조가 고정됨 -> 동적으로 메시지 변경 불가능
```
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
* ✅ NOT_FOUND_POST | ❌ POST_NOT_FOUND
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
* 런타임 맥락에 따라 메시지를 수정하는 클래스 추가
``` java
@Getter
@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
public class DynamicErrorCode implements ErrorCode{

    private final int httpStatus;
    private final String code;
    private final String message;

    // 메시지 변경만 가능함을 명시 (status, code는 변경 불가)
    public static DynamicErrorCode create(ErrorCode errorCode, String message) {
        return new DynamicErrorCode(errorCode.getHttpStatus(), errorCode.getCode(), message);
    }
}

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

### 📍 예외 구조의 재정립 + 줄어든 커스텀 예외 클래스의 성과
* 책임에 따른 ErrorCode의 분리 + 명명 규칙 정립 -> ErrorCode 관리와 Enum 상수 탐색의 용이성
* 중복되는 커스텀 예외 통폐합 -> 커스텀 예외 클래스의 재사용성 증가, 예외 클래스 탐색의 용이성
* 런타임 맥락을 예외 응답에 반영 -> 클라이언트에게 명확한 응답 전달 가능

<br>

****
