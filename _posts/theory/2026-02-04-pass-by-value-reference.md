---
layout: single
title: Pass by Value / Pass by Reference
---

# Pass by Value

<aside>
💡

메서드 파라미터가 인자로부터 값의 복사본을 전달받는 파라미터 전달 매커니즘(parameter passing mechanism).

</aside>

## 특징

- 기본 자료형의 경우 기본 자료형 값의 복사본을, 참조 자료형의 경우 참조값의 복사본을 전달 받는다.
- 메서드 파라미터에 발생하는 변화가 복사본을 제공한 인자에 영향을 주지 않는다.

## 예시

```java
public class Example {
    public static void modifyValue(int x) {
        x = 20;  // This only changes the copy
        System.out.println("Inside method: " + x); // Prints 20
    }
    
    public static void main(String[] args) {
        int num = 10;
        System.out.println("Before: " + num); // Prints 10
        modifyValue(num);
        System.out.println("After: " + num);  // Still prints 10
    }
}
```

# Pass by Reference

<aside>
💡

메서드의 파라미터가 인자 그 자체에 대한 참조를 부여받는 파라미터 전달 매커니즘.

</aside>

## 특징

- 메서드 파라미터의 값에 변화가 가해지면 값을 제공한 인자에 영향을 준다.
    - 메서드의 파라미터로 인자를 재할당하거나 인자의 값을 수정할 수 있다.
- 메서드의 파라미터가 인자에 대한 포인터가 된다.
- Java에서 지원하지 않는 방식이다.

## 예시(C#)

```csharp
using System;

class Program {
    // Pass by reference using 'ref' keyword
    static void PassByReference(ref int x) {
        x = 100;
    }
    
    static void Main() {
        int number = 5;
        
        PassByReference(ref number);
        Console.WriteLine($"After PassByReference: {number}"); // 100
    }
}
```

# Java는 언제나 Pass by Value이다

- Java는 모든 자료형들을 Pass by Value로 처리하지만 기본 자료형과 참조 자료형을 처리하는 방식이 다르다.

## 메서드 파라미터의 값을 처리하는 과정

- 컴파일 타임
    - 컴파일러가 메서드의 범위를 파악한 후 메서드의 범위 내에 선언된 모든 지역 변수에 인덱스를 매긴다.
    - 변수의 이름 대신 인덱스를 사용하도록 바이트코드를 생성한다.
- 런타임
    - 메서드가 실행되면 JVM이 해당 메서드에 대한 Stack Frame을 만들어 Stack 영역에 보관한다.
    - JVM이 인덱스를 통해 Stack Frame 내부에 있는 Local Variable Array에 보관된 지역 변수의 값을 찾아 사용한다.
    - 기본 자료형인 경우 LVA 내에 있는 값을 직접 사용하고, 참고 자료형인 경우 LVA 에 있는 참조값을 이용하여 Heap 영역 내에 있는 객체를 찾아 사용한다.

```csharp
// 메서드 내에 선언된 지역변수들
int count = 10; // 인덱스 1
String name = "hello"; // 인덱스 2
```

## ➕참조값을 전달한다고 Pass by Reference가 아니다

- 질문 배경
    - Pass by Value가 리터럴 값을 전달한다는 의미인 것 같으니 Pass by Reference은 참조값을 전달한다는 의미 같다.
- Pass by Reference는 참조값을 전달한다는 의미가 아니라 **값을 전달한 인자에 대한 직접적인 참조, 즉 접근권**을 전달한다는 의미이다.
- Pass by Reference 기능을 지원하는 C#과 같은 언어에서는 메서드 파라미터가 값을 전달한 인자를 수정할 수 있다.

## ➕할당된 메서드 파라미터가 인자가 가리키고 있는 객체의 내용을 수정할 수 있는데, Pass by Value라고?

- 질문 배경
    - 참조값을 건네받은 변수는 값을 건네준 변수가 가리키고 있는 객체의 내용물을 수정할 수 있다.
    - 이런데도 참조값을 건네받은 변수가 참조값을 제공한 변수에 영향을 주지 않는다고 할 수 있을까?
- 변수가 가리키는 객체를 바꾸는 것과 변수가 가리키는 객체의 내용물을 바꾸는 것은 **다른 행위**다.
- 할당된 메서드 파라미터 참조값을 이용해 값을 전달한 인자가 가리키는 객체의 내용을 수정할 수 있지만, **값을 전달한 인자가 무엇을 가리키는지는 변경할 수 없다**.
- 따라서 Java가 참조값을 할당한다 한들 Pass by Reference가 아닌 Pass by Value라고 하는 것이다.
- 코드로 보는 예시
    - Java가 할 수 있는 것 - 가리키는 객체의 내용물 변경
    
    ```csharp
    MyObject obj1 = new MyObject("A");
    MyObject obj2 = obj1; // Assigning reference value
    obj2.setName("B");     // Modifies the same object that obj1 points to
    System.out.println(obj1.getName()); // Prints "B" - the change is visible!
    ```
    
    - Java가 할 수 없는 것 - 변수가 가리키는 객체 변경
    
    ```csharp
    MyObject obj1 = new MyObject("A");
    MyObject obj2 = obj1; // Both hold the SAME reference value (like two remotes for one TV)
    
    obj2.setName("B");     // ✅ This works - using your remote to change the TV
    obj2 = new MyObject("C"); // ❌ This does NOT affect obj1's reference
    ```
    

## ➕Pass by Value 과정에서 “변수에 값의 복사본이 할당된다”는 어느 과정에서 해당되는 것일까?

- 질문 배경
    - Pass by Value로 변수에 값을 할당할 때 “복사본이 할당된다”는 말은 여러 번 들어봐쓸 것이다. 그렇다면 메서드의 실행 과정에서 언제 “복사본”이 할당되는 과정이 일어날까?
- 개발자의 시점: 개발자의 시점에서 봤을 때 값이 변수에 할당되는 모습으로 보인다.
- JVM의 시점: Stack 영역에 저장되는 Stack Frame을 생성할 때 Local Variable Array의 요소를 소스 코드의 값을 보고 만든다.
- 개발자의 시점이든 JVM의 시점이든 **메서드 파라미터에 수정이 가해져도 값을 전달한 인자가 받는 영향은 없다**는 점에서 “변수에 복사본을 할당한다”고 하는 것이다.

## ➕메모리에 저장되는 요소와 저장되지 않는 요소

- 저장되는 요소
    - 실제 값(10, 3.14, “Hello, World!”)
    - 메모리 위치(스택 슬롯, 힙의 주소)
    - 클래스 구조와 메서드의 정의(메서드 영역 내에 저장함)
    - 문자열 리터럴(String Pool에 저장)
- 저장되지 않는 요소
    - 개발자와 컴파일러가 사용하는 요소들이다.
    - 키워드(int, class, public 등)
    - 변수 이름
    - 메서드 이름
    - 소스 코드의 구조(공백, 주석 등)
